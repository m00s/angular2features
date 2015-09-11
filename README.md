## Sources
* [Official documentation](https://github.com/angular/angular/tree/master/modules/angular2/docs)
* [Viktor Savkin's articles](http://victorsavkin.com/)

# Angular 2 features

* [Annotations](#annotations)
  * [Directives](#directives)
  * [Components](#components)
* [View](#view)
  * [Overview](#overview)
  * [Simple View](#simple-view)
  * [Composed View](#composed-view)
  * [Component Views](#component-view)
  * [Evaluation Context](#evaluation-context)
  * [Lifecycle (Hydration and Dehydration)](#lifecycle-hydration-and-dehydration)
* [Templates](#templates)
  * [Property bindings](#property-bindings)
  * [Binding events](#binding-events)
  * [String Interpolation](#string-interpolation)
  * [Inline Templates](#inline-templates)
  * [Template Microsyntax](#template-microsyntax)
* [Change detection](#change-detection)
  * [Immutable Objects](#immutable-objects)
  * [Observable Objects](#observable-objects)
* [DI](#di)
  * [Core Abstractions](#core-abstractions)
  * [Example](#example)
  * [Child Injectors and Dependencies](#child-injectors-and-dependencies)
    * [Constraints](#constraints)
    * [DI Does Not Walk Down](#di-does-not-walk-down)
  * [Bindings](#bindings)
    * [Resolved Bindings](#resolved-bindings)
  * [Transient Dependencies](#transient-dependencies)
* [HTTP](#http)
* [Pipes](#pipes)
* [Router](#router)
* [Test](#test)
* [Web Workers](#web-workers)

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
### View
-----------------------------------
#### Overview
A View is a core primitive used by angular to render the DOM tree.
A ViewContainer is location in a View which can accept child Views.
Every ViewContainer has an associated ViewContainerRef than can contain any number of child Views.
Views form a tree structure which mimics the DOM tree.

* View is a core rendering construct. A running application is just a collection of Views which are
  nested in a tree like structure. The View tree is a simplified version of the DOM tree. A View can
  have a single DOM Element or large DOM structures. The key is that the DOM tree in the View can
  not undergo structural changes (only property changes).
* Views represent a running instance of a DOM View. This implies that while elements in a View
  can change properties, they can not change structurally. (Structural changes such as, adding or
  removing elements requires adding or removing child Views into ViewContainers).
* View can have zero or more ViewContainers. A ViewContainer is a marker in the DOM which allows
  the insertion of child Views.
* Views are created from a ProtoView. A ProtoView is a compiled DOM View which is efficient at
  creating Views.
* View contains a context object. The context represents the object instance against which all
  expressions are evaluated.
* View contains a ChangeDetector for looking for detecting changes to the model.
* View contains ElementInjector for creating Directives.

#### Simple View

Let's examine a simple View and all of its parts in detail.

Assume the following Component:

``` javascript
class Greeter {
  greeting:string;

  constructor() {
    this.greeting = 'Hello';
  }
}
```

And assume following HTML View:

``` html
<div>
  Your name:
  <input var="name" type="Text">
  <br>
  {{greeting}} {{name.value}}!
</div>
```

The above template is compiled by the Compiler to create a ProtoView. The ProtoView is then used to
create an instance of the View. The instantiation process involves cloning the above template and
locating all of the elements which contain bindings and finally instantiating the Directives
associated with the template. (See compilation for more details.)

``` html
<div>                             | viewA(greeter)
  Your name:                      | viewA(greeter)
  <input var="name" type="Text">  | viewA(greeter): local variable 'name'
  <br>                            | viewA(greeter)
  {{greeting}} {{name.value}}!    | viewA(greeter): binding expression 'greeting' & 'name.value'
</div>                            | viewA(greeter)
```

The resulting View instance looks something like this (simplified pseudo code):
``` javascript
viewA = new View({
  template: ...,
  context: new Greeter(),
  localVars: ['name'],
  watchExp: ['greeting', 'name.value']
});
```

Note:
* View uses instance of `Greeter` as the evaluation context.
* View knows of local variables `name`.
* View knows which expressions need to be watched.
* View knows what needs to be updated if the watched expression changes.
* All DOM elements are owned by single instance of the view.
* The structure of the DOM can not change during runtime. To allow structural changes to the DOM we need
  to understand Composed View.


#### Composed View

An important part of an application is to be able to change the DOM structure to render data for the
user. In Angular this is done by inserting child views into the ViewContainer.

Let's start with a View such as:

``` html
<ul>
  <li template="ng-for: #person of people">{{person}}</li>
</ul>
```

During the compilation process the Compiler breaks the HTML template into these two ProtoViews:

``` html
  <li>{{person}}</li>   | protoViewB(Locals)
```

and

``` html
<ul>                    | protoViewA(someContext)
  <template></template> | protoViewA(someContext): protoViewB
</ul>                   | protoViewA(someContext)
```


The next step is to compose these two ProtoViews into an actual view which is rendered to the user.

*Step 1:* Instantiate `viewA`

``` html
<ul>                    | viewA(someContext)
  <template></template> | viewA(someContext): new NgFor(new ViewContainer(protoViewB))
</ul>                   | viewA(someContext)
```

*Step2:* Instantiate `NgFor` directive which will receive the `ViewContainerRef`. (The ViewContainerRef
has a reference to `protoViewA`).


*Step3:* As the `NgFor` directive unrolls it asks the `ViewContainerRef` to instantiate `protoViewB` and insert
it after the `ViewContainer` anchor. This is repeated for each `person` in `people`. Notice that

``` html
<ul>                    | viewA(someContext)
  <template></template> | viewA(someContext): new NgFor(new ViewContainer(protoViewB))
  <li>{{person}}</li>   | viewB0(locals0(someContext))
  <li>{{person}}</li>   | viewB1(locals0(someContext))
</ul>                   | viewA(someContext)
```

*Step4:* All of the bindings in the child Views are updated. Notice that in the case of `NgFor`
the evaluation context for the `viewB0` and `viewB1` are `locals0` and `locals1` respectively.
Locals allow the introduction of new local variables visible only within the scope of the View, and
delegate any unknown references to the parent context.

``` html
<ul>                    | viewA
  <template></template> | viewA: new NgFor(new ViewContainer(protoViewB))
  <li>Alice</li>        | viewB0
  <li>Bob</li>          | viewB1
</ul>                   | viewA
```

Each View can have zero or more ViewContainers. By inserting and removing child Views to and from the
ViewContainers, the application can mutate the DOM structure to any desirable state. A View may contain
individual nodes or a complex DOM structure. The insertion points for the child Views, known as
ViewContainers, contain a DOM element which acts as an anchor. The anchor is either a `template` or
a `script` element depending on your browser. It is used to identify where the child Views will be
inserted.

#### Component Views

A View can also contain Components. Components contain Shadow DOM for encapsulating their internal
rendering state. Unlike ViewContainers which can contain zero or more Views, the Component always contains
exactly one Shadow View.

``` html
<div>                            | viewA
  <my-component>                 | viewA
    #SHADOW_ROOT                 | (encapsulation boundary)
      <div>                      |   viewB
         encapsulated rendering  |   viewB
      </div>                     |   viewB
  </my-component>                | viewA
</div>                           | viewA
```

#### Evaluation Context

Each View acts as a context for evaluating its expressions. There are two kinds of contexts:

1. A component controller instance and
2. a `Locals` context for introducing local variables into the View.

Let's assume following component:

``` javascript
class Greeter {
  greeting:string;

  constructor() {
    this.greeting = 'Hello';
  }
}
```

And assume the following HTML View:

``` html
<div>                             | viewA(greeter)
  Your name:                      | viewA(greeter)
  <input var="name" type="Text">  | viewA(greeter)
  <br>                            | viewA(greeter)
  {{greeting}} {{name.value}}!    | viewA(greeter)
</div>                            | viewA(greeter)
```

The above UI is built using a single View, and hence a single context `greeter`. It can be expressed
in this pseudo-code.

``` javascript
var greeter = new Greeter();
```

The View contains two bindings:

1. `greeting`: This is bound to the `greeting` property on the `Greeter` instance.
2. `name.value`: This poses a problem. There is no `name` property on the `Greeter` instance. To solve
this we wrap the `Greeter` instance in the `Local` instance like so:

``` javascript
var greeter = new Locals(new Greeter(), {name: ref_to_input_element })
```


By wrapping the `Greeter` instance into the `Locals` we allow the view to introduce variables which
are in addition to the `Greeter` instance. During the resolution of the expressions we first check
the locals, and then the `Greeter` instance.



#### View LifeCycle (Hydration and Dehydration)

Views transition through a particular set of states:

1. View is created from the ProtoView.
2. View can be attached to an existing ViewContainerRef.
3. Upon attaching View to the ViewContainerRef the View needs to be hydrated. The hydration process
   involves instantiating all of the Directives associated with the current View.
4. At this point the view is ready and renderable. Multiple changes can be delivered to the
   Directives from the ChangeDetection.
5. At some point the View can be removed. At this point all of the directives are destroyed during
   the dehydration process and the view becomes inactive.
6. The View has to wait until it is detached from the DOM. The delay in detaching could be caused
   because an animation is animating the view away.
7. After the View is detached from the DOM it is ready to be reused. The view reuse allows the
   application to be faster in subsequent renderings.


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
``` html
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

#### Binding Events

Binding events allows wiring events from DOM (or other components) to the Angular controller.

**Short form:** `<some-element (some-event)="statement">`

**Canonical form:** `<some-element on-some-event="statement">`

Where:
* `some-element` Any element which can generate DOM events (or has an angular directive which generates the event).
* `some-event` (escaped with `()` or `on-`) is the name of the event `some-event`. In this case the
  dash-case is converted into camel-case `someEvent`.
* `statement` is a valid statement (as defined in section below).
If the execution of the statement returns `false`, then `preventDefault`is applied on the DOM event.

Angular listens to bubbled DOM events (as in the case of clicking on any child), as shown below:

**Short form:** `<some-element (some-event)="statement">`

**Canonical form:** `<some-element on-some-event="statement">`

Example:
``` js
@Component(...)
class Example {
  submit() {
    // do something when button is clicked
  }
}

<button (click)="submit()">Submit</button>
```

In the above example, when clicking on the submit button angular will invoke the `submit` method on the surrounding
component's controller.


**NOTE:** Unlike Angular v1, Angular v2 treats event bindings as core constructs not as directives. This means that there
is no need to create a event directive for each kind of event. This makes it possible for Angular v2 to easily
bind to custom events of Custom Elements, whose event names are not known ahead of time.

#### String Interpolation

Property bindings are the only data bindings which Angular supports, but for convenience Angular supports an interpolation
syntax which is just a short hand for the data binding syntax.

``` html
<span>Hello {{name}}!</span>
```

is a short hand for:

``` html
<span [text|0]=" 'Hello ' + stringify(name) + '!'">_</span>
```

The above says to bind the `'Hello ' + stringify(name) + '!'` expression to the zero-th child of the `span`'s `text`
property. The index is necessary in case there are more than one text nodes, or if the text node we wish to bind to
is not the first one.

Similarly the same rules apply to interpolation inside attributes.

``` html
<span title="Hello {{name}}!"></span>
```

is a short hand for:

``` html
<span [title]=" 'Hello ' + stringify(name) + '!'"></span>
```

**NOTE:** `stringify()` is a built in implicit function which converts its argument to a string representation, while
keeping `null` and `undefined` as empty strings.

#### Inline Templates
Data binding allows updating the DOM's properties, but it does not allow for changing of the DOM structure. To change
DOM structure we need the ability to define child templates, and then instantiate these templates into Views. The
Views than can be inserted and removed as needed to change the DOM structure.

**Short form:**
``` html
Hello {{user}}!
<div template="ng-if: isAdministrator">
  ...administrator menu here...
</div>
```

**Canonical form:**
``` html
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

``` html
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

``` html
<ul>
  <li template="ng-for; #person; of=people; #i=index;">{{i}}. {{person}}<li>
</ul>
```

Notice how each key value pair is translated to a `key=value;` statement in the `template` attribute. This makes the
repeat syntax a much shorter, but we can do better. Turns out that most punctuation is optional in the short version
which allows us to further shorten the text.

``` html
<ul>
  <li template="ng-for #person of people #i=index">{{i}}. {{person}}<li>
</ul>
```

We can also optionally use `var` instead of `#` and add `:` to `for` which creates the following recommended
microsyntax for `ng-for`.

``` html
<ul>
  <li template="ng-for: var person of people; var i=index">{{i}}. {{person}}<li>
</ul>
```

Finally, we can move the `ng-for` keyword to the left hand side and prefix it with `*` as so:

``` html
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

![Change detection](http://36.media.tumblr.com/70d4551eef20b55b195c3232bf3d4e1b/tumblr_njb2puhhEa1qc0howo2_1280.png)
Every component gets a change detector responsible for checking the bindings defined in its template. Examples of bindings: `{{todo.text}}` and `[todo]="t"`. Change detectors propagate bindings[c] from the root to leaves in the depth first order.

By default the change detection goes through every node of the tree to see if it changed, and it does it on every browser event. Although it may seem terribly inefficient, the Angular 2 change detection system can go through hundreds of thousands of simple checks (the number are platform dependent) in a few milliseconds.

#### Immutable Objects
If a component depends only on its bindings, and the bindings are immutable, then this component can change if and only if one of its bindings changes. Therefore, we can skip the component’s subtree in the change detection tree until such an event occurs. When it happens, we can check the subtree once, and then disable it until the next change (gray boxes indicate disabled change detectors).
![CD - Immutable Objects](http://40.media.tumblr.com/0f43874fd6b8967f777ac9384122b589/tumblr_njb2puhhEa1qc0howo4_1280.png)

To implement this just set the change detection strategy to `ON_PUSH`.

``` javascript
@Component({changeDetection:ON_PUSH})
class ImmutableTodoCmp {
  todo:Todo;
}
```

#### Observable Objects
If a component depends only on its bindings, and the bindings are observable, then this component can change if and only if one of its bindings emits an event. Therefore, we can skip the component’s subtree in the change detection tree until such an event occurs. When it happens, we can check the subtree once, and then disable it until the next change.

**NOTE:** If you have a tree of components with immutable bindings, a change has to go through all the components starting from the root. This is not the case when dealing with observables.

``` javascript
type ObservableTodo = Observable<Todo>;
type ObservableTodos = Observable<Array<ObservableTodo>>;

@Component({selector:’todos’})
class ObservableTodosCmp {
  todos:ObservableTodos;
  //...
}
```

``` javascript
<todo *ng-for=”var t of todos” todo=”t”></todo>
```

``` javascript
@Component({selector:'todo'})
class ObservableTodoCmp {
  todo:ObservableTodo;
  //...
}
```

Say the application uses only observable objects. When it boots, Angular will check all the objects.

![CD - Ob 1](http://40.media.tumblr.com/b9a743a15d23c3db9f910f4c7566b928/tumblr_njb2puhhEa1qc0howo5_1280.png)

After the first pass will look as follows:

![CD - Ob 2](http://40.media.tumblr.com/5f4ba2e53fb3de05f9c199199f4aae77/tumblr_njb2puhhEa1qc0howo6_1280.png)

Let’s say the first todo observable fires an event. The system will switch to the following state:

![CD - Ob 3](http://40.media.tumblr.com/cb54aedb3479e1b0578ae2c6c8c7ccc2/tumblr_njb2puhhEa1qc0howo7_1280.png)

And after checking `App_ChangeDetector`, `Todos_ChangeDetector`, and the first `Todo_ChangeDetector`, it will go back to this state.

![CD - Ob 4](http://40.media.tumblr.com/5f4ba2e53fb3de05f9c199199f4aae77/tumblr_njb2puhhEa1qc0howo6_1280.png)

Assuming that changes happen rarely and the components form a balanced tree, using observables changes the complexity of change detection from `O(N)` to `O(logN)`, where N is the number of bindings in the system.

**REMEMBER**
- An Angular 2 application is a reactive system.
- The change detection system propagates bindings from the root to leaves.
- Unlike Angular 1.x, the change detection graph is a directed tree. As a result, the system is more performant and predictable.
- By default, the change detection system walks the whole tree. But if you use immutable objects or observables, you can take advantage of them and check parts of the tree only if they "really change".
- These optimizations compose and do not break the guarantees the change detection provides.

<!-- -------------------------- -->
### DI
-----------------------------------

#### Core Abstractions

The library is built on top of the following core abstractions: `Injector`, `Binding`, and `Dependency`.

 * An injector is created from a set of bindings.
 * An injector resolves dependencies and creates objects.
 * A binding maps a token, such as a string or class, to a factory function and a list of dependencies. So a binding defines how to create an object.
 * A dependency points to a token and contains extra information on how the object corresponding to that token should be injected.

```
[Injector]
    |
    |
    |*
[Binding]
   |----------|-----------------|
   |          |                 |*
[Token]    [FactoryFn]     [Dependency]
                               |---------|
                               |         |
                            [Token]   [Flags]
```

#### Example

``` javascript
class Engine {
}

class Car {
	constructor(@Inject(Engine) engine) {
	}
}

var inj = Injector.resolveAndCreate([
	bind(Car).toClass(Car),
	bind(Engine).toClass(Engine)
]);
var car = inj.get(Car);
```

In this example we create two bindings: one for Car and one for Engine. `@Inject(Engine)` declares a dependency on Engine.



#### Injector

An injector instantiates objects lazily, only when asked for, and then caches them.

Compare

``` javascript
var car = inj.get(Car); //instantiates both an Engine and a Car
```

with

``` javascript
var engine = inj.get(Engine); //instantiates an Engine
var car = inj.get(Car); //instantiates a Car (reuses Engine)
```

and with

``` javascript
var car = inj.get(Car); //instantiates both an Engine and a Car
var engine = inj.get(Engine); //reads the Engine from the cache
```

To avoid bugs make sure the registered objects have side-effect-free constructors. In this case, an injector acts like a hash map, where the order in which the objects got created does not matter.


#### Child Injectors and Dependencies

Injectors are hierarchical.

``` js
var parent = Injector.resolveAndCreate([
  bind(Engine).toClass(TurboEngine)
]);
var child = parent.resolveAndCreateChild([Car]);

var car = child.get(Car); // uses the Car binding from the child injector and Engine from the parent injector.
```

Injectors form a tree.

```
  GrandParentInjector
   /              \
Parent1Injector  Parent2Injector
  |
ChildInjector
```

The dependency resolution algorithm works as follows:

``` js
// this is pseudocode.
var inj = this;
while (inj) {
  if (inj.hasKey(requestedKey)) {
    return inj.get(requestedKey);
  } else {
    inj = inj.parent;
  }
}
throw new NoBindingError(requestedKey);
```

So in the following example

``` js
class Car {
  constructor(e: Engine){}
}
```

DI will start resolving `Engine` in the same injector where the `Car` binding is defined. It will check whether that injector has the `Engine` binding. If it is the case, it will return that instance. If not, the injector will ask its parent whether it has an instance of `Engine`. The process continues until either an instance of `Engine` has been found, or we have reached the root of the injector tree.

##### Constraints

You can put upper and lower bound constraints on a dependency. For instance, the `@Self` decorator tells DI to look for `Engine` only in the same injector where `Car` is defined. So it will not walk up the tree.

``` js
class Car {
  constructor(@Self() e: Engine){}
}
```

A more realistic example is having two bindings that have to be provided together (e.g., NgModel and NgRequiredValidator.)

The `@Host` decorator tells DI to look for `Engine` in this injector, its parent, until it reaches a host (see the section on hosts.)

``` js
class Car {
  constructor(@Host() e: Engine){}
}
```

The `@SkipSelf` decorator tells DI to look for `Engine` in the whole tree starting from the parent injector.

``` js
class Car {
  constructor(@SkipSelf() e: Engine){}
}
```

###### DI Does Not Walk Down


Dependency resolution only walks up the tree. The following will throw because DI will look for an instance of `Engine` starting from `parent`.

``` js
var parent = Injector.resolveAndCreate([Car]);
var child = parent.resolveAndCreateChild([
  bind(Engine).toClass(TurboEngine)
]);

parent.get(Car); // will throw NoBindingError
```


#### Bindings

You can bind to a class, a value, or a factory. It is also possible to alias existing bindings.

``` js
var inj = Injector.resolveAndCreate([
  bind(Car).toClass(Car),
  bind(Engine).toClass(Engine)
]);

var inj = Injector.resolveAndCreate([
  Car,  // syntax sugar for bind(Car).toClass(Car)
  Engine
]);

var inj = Injector.resolveAndCreate([
  bind(Car).toValue(new Car(new Engine()))
]);

var inj = Injector.resolveAndCreate([
  bind(Car).toFactory((e) => new Car(e), [Engine]),
  bind(Engine).toFactory(() => new Engine())
]);
```

You can bind any token.

``` js
var inj = Injector.resolveAndCreate([
  bind(Car).toFactory((e) => new Car(), ["engine!"]),
  bind("engine!").toClass(Engine)
]);
```

If you want to alias an existing binding, you can do so using `toAlias`:

``` js
var inj = Injector.resolveAndCreate([
  bind(Engine).toClass(Engine),
  bind("engine!").toAlias(Engine)
]);
```
which implies `inj.get(Engine) === inj.get("engine!")`.

Note that tokens and factory functions are decoupled.

``` js
bind("some token").toFactory(someFactory);
```

The `someFactory` function does not have to know that it creates an object for `some token`.

##### Resolved Bindings

When DI receives `bind(Car).toClass(Car)`, it needs to do a few things before before it can create an instance of `Car`:

- It needs to reflect on `Car` to create a factory function.
- It needs to normalize the dependencies (e.g., calculate lower and upper bounds).

The result of these two operations is a `ResolvedBinding`.

The `resolveAndCreate` and `resolveAndCreateChild` functions resolve passed-in bindings before creating an injector. But you can resolve bindings yourself using `Injector.resolve([bind(Car).toClass(Car)])`. Creating an injector from pre-resolved bindings is faster, and may be needed for performance sensitive areas.

You can create an injector using a list of resolved bindings.

``` js
var listOfResolvingBindings = Injector.resolve([Binding1, Binding2]);
var inj = Injector.fromResolvedBindings(listOfResolvingBindings);
inj.createChildFromResolvedBindings(listOfResolvedBindings);
```


##### Transient Dependencies

An injector has only one instance created by each registered binding.

``` js
inj.get(MyClass) === inj.get(MyClass); //always holds
```

If we need a transient dependency, something that we want a new instance of every single time, we have two options.

We can create a child injector for each new instance:

``` js
var child = inj.resolveAndCreateChild([MyClass]);
child.get(MyClass);
```

Or we can register a factory function:

``` js
var inj = Injector.resolveAndCreate([
  bind('MyClassFactory').toFactory(dep => () => new MyClass(dep), [SomeDependency])
]);

var factory = inj.get('MyClassFactory');
var instance1 = factory(), instance2 = factory();
// Depends on the implementation of MyClass, but generally holds.
expect(instance1).not.toBe(instance2);
```
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
<!-- -------------------------- -->
### Test
-----------------------------------
### Web Workers

Angular 2 includes native support for writing applications which live in a
WebWorker. This document describes how to write applications that take advantage
of this feature.
It also provides a detailed description of the underlying messaging
infrastructure that angular uses to communicate between the main process and the
worker. This infrastructure can be modified by an application developer to
enable driving an angular 2 application from an iFrame, different window / tab,
server, etc..

#### Introduction
WebWorker support in Angular2 is designed to make it easy to leverage parallelization in your web application.
When you choose to run your application in a WebWorker angular runs both your application's logic and the 
majority of the core angular framework in a WebWorker. 
By offloading as much code as possible to the WebWorker we keep the UI thread
free to handle events, manipulate the DOM, and run animations. This provides a
better framerate and UX for applications.

#### Bootstrapping a WebWorker Application
Bootstrapping a WebWorker application is not much different than bootstrapping a normal application. 
The primary difference is that you don't pass your root component directly to ```bootstrap```. 
Instead you pass the name of a background script that calls ```bootstrapWebWorker``` with your root component.

##### Example
To bootstrap Hello World in a WebWorker we do the following in TypeScript
```HTML
<html>
  <head>
     <script src="https://github.jspm.io/jmcriffey/bower-traceur-runtime@0.0.87/traceur-runtime.js"></script>
     <script src="https://jspm.io/system@0.16.js"></script>
     <script src="angular2/web_worker/ui.js"></script>
  </head>
  <body>
    <hello-world></hello-world>
    <script>System.import("index")</script>
  </body>
</html>
```
```TypeScript
// index.js
import {bootstrap} from "angular2/web_worker/ui";
bootstrap("loader.js");
```
```JavaScript
// loader.js
importScripts("https://github.jspm.io/jmcriffey/bower-traceur-runtime@0.0.87/traceur-runtime.js", "https://jspm.io/system@0.16.js", "angular2/web_worker/worker.js");
System.import("app");
```
```TypeScript
// app.ts
import {Component, View, bootstrapWebWorker} from "angular2/web_worker/worker";
@Component({
  selector: "hello-world"
})
@View({
  template: "<h1>Hello {{name}}</h1>
})
export class HelloWorld {
  name: string = "Jane";
}

bootstrapWebWorker(HelloWorld);
```
There's a few important things to note here:
* On the UI side we import all angular types from `angular2/web_worker/ui` and on the worker side we import from
`angular2/web_worker/worker`. These modules include all the typings in the WebWorker bundle. By importing from
these URLs instead of `angular2/angular2` we can statically ensure that our app does not reference a type that
doesn't exist in the context it's mean to execute in. For example, if we tried to import DomRenderer in the Worker
or NgFor on the UI we would get a compiler error.
* The UI loads angular from the file `angular2/web_worker/ui.js` and the Worker loads angular from
`angular2/web_worker/worker.js`. These bundles are created specifically for using WebWorkers and should be used
instead of the normal angular2.js file. Both files contain subsets of the angular2 codebase that is designed to
run specifically on the UI or Worker. Additionally, they contain the core messaging infrastructure used to
communicate between the Worker and the UI. This messaging code is not in the standard angular2.js file.
* We pass `loader.js` to bootstrap and not `app.ts`. You can think of `loader.js` as the `index.html` of the
Worker. Since WebWorkers share no memory with the UI we need to reload the angular2 dependencies before
bootstrapping our application. We do this with importScripts. Additionally, we need to do this in a different 
file than `app.ts` because our module loader (System.js in this example) has not been loaded yet, and `app.ts`
will be compiled with a System.define call at the top.
* The HelloWorld Component looks exactly like a normal Angular2 HelloWorld Component! The goal of WebWorker
support was to allow as much of Angular to live in the worker as possible. 
As such, *most* angular2 components can be bootstrapped in a WebWorker with minimal to no changes required.

For reference, here's the same HelloWorld example in Dart.
```HTML
<html>
  <body>
    <script type="application/dart" src="index.dart"></script>
    <script src="packages/browser.dart.js"></script>
  </body>
</html>
```
```Dart
// index.dart
import "package:angular2/web_worker/ui.dart";
import "package:angular2/src/core/reflection/reflection.dart";
import "package:angular2/src/core/reflection/reflection_capabilities.dart";

main() {
  reflector.reflectionCapabilities = new ReflectionCabilities();
  bootstrap("app.dart");
}
```
```Dart
import "package:angular2/web_worker/worker.dart";
import "package:angular2/src/core/reflection/reflection.dart";
import "package:angular2/src/core/reflection/reflection_capabilities.dart";

@Component(
  selector: "hello-world"
)
@View(
  template: "<h1>Hello {{name}}</h1>"
)
class HelloWorld {
  String name = "Jane";
}

main(List<String> args, SendPort replyTo) {
  reflector.reflectionCapabilities = new ReflectionCapabilities();
  bootstrapWebWorker(replyTo, HelloWorld);
}

```
This code is nearly the same as the TypeScript version with just a couple key differences:
* We don't have a `loader.js` file. Dart applications don't need this file because you don't need a module loader.
* We pass a `SendPort` to `bootstrapWebWorker`. Dart applications use the Isolate API, which communicates via
Dart's Port abstraction. When you call `bootstrap` from the UI thread, angular starts a new Isolate to run 
your application logic. When Dart starts a new Isolate it passes a `SendPort` to that Isolate so that it 
can communicate with the Isolate that spawned it. You need to pass this `SendPort` to `bootstrapWebWorker` 
so that Angular can communicate with the UI.
* You need to set up `ReflectionCapabilities` on both the UI and Worker. Just like writing non-concurrent 
Angular2 Dart applications you need to set up the reflector. You should not use Reflection in production, 
but should use the angular 2 transformer to remove it in your final JS code. Note there's currently a bug 
with running the transformer on your UI code (#3971). You can (and should) pass the file where you call
`bootstrapWebWorker` as an entry point to the transformer, but you should not pass your UI index file 
to the transformer until that bug is fixed.

#### Writing WebWorker Compatible Components
You can do almost everything in a WebWorker component that you can do in a typical Angular 2 Component. 
The main exception is that there is **no** DOM access from a WebWorker component. In Dart this means you can't 
import anything from `dart:html` and in JavaScript it means you can't use `document` or `window`. Instead you 
should use data bindings and if needed you can inject the `Renderer` along with your component's `ElementRef`
directly into your component and use methods such as `setElementProperty`, `setElementAttribute`,
`setElementClass`, `setElementStyle`, `invokeElementMethod`, and `setText`. Not that you **cannot** call
`getNativeElementSync`. Doing so will always return `null` when running in a WebWorker. 
If you need DOM access see [Running Code on the UI](#running-code-on-the-ui).

#### WebWorker Design Overview
When running your application in a WebWorker, the majority of the angular core along with your application logic
runs on the worker. The two main components that run on the UI are the `Renderer` and the `RenderCompiler`. When
running angular in a WebWorker the bindings for these two components are replaced by the `WebWorkerRenderer` and 
the `WebWorkerRenderCompiler`. When these components are used at runtime, they pass messages through the 
[MessageBroker](#messagebroker) instructing the UI to run the actual method and return the result. The
[MessageBroker](#messagebroker) abstraction allows either side of the WebWorker boundary to schedule code to run
on the opposite side and receive the result. You can use the [MessageBroker](#messagebroker)
Additionally, the [MessageBroker](#messagebroker) sits on top of the [MessageBus](#messagebus). 
MessageBus is a low level abstraction that provides a language agnostic API for communicating with angular components across any runtime boundary such as `WebWorker <--> UI` communication, `UI <--> Server` communication, 
or `Window <--> Window` communication.

See the diagram below for a high level overview of how this code is structured:

![WebWorker Diagram](http://stanford.edu/~jteplitz/ng_2_worker.png)

#### Running Code on the UI
If your application needs to run code on the UI, there are a few options. The easiest way is to use a 
CustomElement in your view. You can then register this custom element from your html file and run code in response
to the element's lifecycle hooks. Note, Custom Elements are still experimental. See 
[MDN](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Custom_Elements) for the latest details on how
to use them.

If you require more robust communication between the WebWorker and the UI you can use the [MessageBroker](#using-the-messagebroker-in-your-application) or
[MessageBus](#using-the-messagebus-in-your-application) directly.

#### MessageBus
The MessageBus is a low level abstraction that provides a language agnostic API for communicating with angular components across any runtime boundary. It supports multiplex communication through the use of a channel
abstraction.

Angular currently includes two stable MessageBus implementations, which are used by default when you run your
application inside a WebWorker.

1. The `PostMessageBus` is used by JavaScript applications to communicate between a WebWorker and the UI.
2. The `IsolateMessageBus` is used by Dart applications to communicate between a background Isolate and the UI.

Angular also includes three experimental MessageBus implementations:

1. The `WebSocketMessageBus` is a Dart MessageBus that lives on the UI and communicates with an angular
application running on a server. It's intended to be used with either the `SingleClientServerMessageBus` or the
`MultiClientServerMessageBus`.
2. The `SingleClientServerMessageBus` is a Dart MessageBus that lives on a Dart Server. It allows an angular
application to run on a server and communicate with a single browser that's running the `WebSocketMessageBus`.
3. The `MultiClientServerMessageBus` is like the `SingleClientServerMessageBus` except it allows an arbitrary
number of clients to connect to the server. It keeps all connected browsers in sync and if an event fires in
any connected browser it propagates the result to all connected clients. This can be especially useful as a 
debugging tool, by allowing you to connect multiple browsers / devices to the same angular application, 
change the state of that application, and ensure that all the clients render the view correctly. Using these tools
can make it easy to catch tricky browser compatibility issues.

##### Using the MessageBus in Your Application
**Note**: If you want to pass custom messages between the UI and WebWorker, it's recommended you use the
[MessageBroker](#using-the-messagebroker-in-your-application). However, if you want to control the messaging
protocol yourself you can use the MessageBus directly.

To use the MessageBus you need to initialize a new channel on both the UI and WebWorker.
In TypeScript that would look like this:
```TypeScript
// index.ts, which is running on the UI.
var instance = bootstrap("loader.js");
var bus = instance.bus;
bus.initChannel("My Custom Channel");
```
```TypeScript
// background_index.ts, which is running on the WebWorker
import {MessageBus} from 'angular2/web_worker/worker';
@Component({...})
@View({...})
export class MyComponent {
  constructor (bus: MessageBus) {
    bus.initChannel("My Custom Channel");
  }
}
```

Once the channel has been initialized either side can use the `from` and `to` methods on the MessageBus to send
and receive messages. Both methods return EventEmitter. Expanding on the example from earlier:
```TypeScript
// index.ts, which is running on the UI.
import {bootstrap} from 'angukar2/web_worker/ui';
var instance = bootstrap("loader.js");
var bus = instance.bus;
bus.initChannel("My Custom Channel");
bus.to("My Custom Channel").next("hello from the UI");
```
```TypeScript
// background_index.ts, which is running on the WebWorker
import {MessageBus, Component, View} from 'angular2/web_worker/worker';
@Component({...})
@View({...})
export class MyComponent {
  constructor (bus: MessageBus) {
    bus.initChannel("My Custom Channel");
    bus.from("My Custom Channel").observer((message) => {
      console.log(message); // will print "hello from the UI"
    });
  }
}
```

This example is nearly identical in Dart, and is included below for reference:
```Dart
// index.dart, which is running on the UI.
import 'package:angular2/web_workers/ui.dart';

main() {
  var instance = bootstrap("background_index.dart");
  var bus = instance.bus;
  bus.initChannel("My Custom Channel");
  bus.to("My Custom Channel").add("hello from the UI");
}

```
```Dart
// background_index.dart, which is running on the WebWorker
import 'package:angular2/web_worker/worker.dart';
@Component(...)
@View(...)
class MyComponent {
  MyComponent (MessageBus bus) {
    bus.initChannel("My Custom Channel");
    bus.from("My Custom Channel").listen((message) {
      print(message); // will print "hello from the UI"
    });
  }
}
```
The only substantial difference between these APIs in Dart and TypeScript is the different APIs for the
`EventEmitter`.

**Note** Because the messages passed through the MessageBus cross a WebWorker boundary, they must be serializable.
If you use the MessageBus directly, you are responsible for serializing your messages.
In JavaScript / TypeScript this means they must be serializable via JavaScript's 
[structured cloning algorithim](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm).

In Dart this means they must be valid messages that can be passed through a
[SendPort](https://api.dartlang.org/1.12.1/dart-isolate/SendPort/send.html).


##### MessageBus and Zones
The MessageBus API includes support for [zones](http://www.github.com/angular/zone.js). 
A MessageBus can be attached to a specific zone  (by calling `attachToZone`). Then specific channels can be
specified to run in the zone when they are initialized.
If a channel is running in the zone, that means that any events emitted from that channel will be executed within
the given zone. For example, by default angular runs the EventDispatch channel inside the angular zone. That means
when an event is fired from the DOM and received on the WebWorker the event handler automatically runs inside the
angular zone. This is desired because after the event handler exits we want to exit the zone so that we trigger
change detection. Generally, you want your channels to run inside the zone unless you have a good reason for why 
they need to run outside the zone.

##### Implementing and Using a Custom MessageBus
**Note** Implementing and using a Custom MesageBus is experimental and requires importing from private APIs.

If you want to drive your application from something other than a WebWorker you can implement a custom message
bus. Implementing a custom message bus just means creating a class that fulfills the API specified by the
abstract MessageBus class.

If you're implementing your MessageBus in Dart you can extend the `GenericMessageBus` class included in angular.
if you do this, you don't need to implement zone or channel support yourself. You only need to implement a 
`MessageBusSink` that extends `GenericMessageBusSink` and a `MessageBusSource` that extends 
`GenericMessageBusSource`. The `MessageBusSink` must override the `sendMessages` method. This method is
given a list of serialized messages that it is required to send through the sink.
The `MessageBusSource` needs to provide a [Stream](https://api.dartlang.org/1.12.1/dart-async/Stream-class.html)
of incoming messages (either by passing the stream to `GenericMessageBusSource's` constructor or by calling
attachTo() with the stream). It also needs to override the abstract `decodeMessages` method. This method is
given a List of serialized messages received by the source and should perform any decoding work that needs to be
done before the application can read the messages.

For example, if your MessageBus sends and receives JSON data you would do the following:
```Dart
import 'package:angular2/src/web_workers/shared/generic_message_bus.dart';
import 'dart:convert';

class JsonMessageBusSink extends GenericMessageBusSink {
  @override
  void sendMessages(List<dynamic> messages) {
    String encodedMessages = JSON.encode(messages);
    // Send encodedMessages here
  }
}

class JsonMessageBusSource extends GenericMessageBuSource {
  JsonMessageBusSource(Stream incomingMessages) : super (incomingMessages);
  
  @override
  List<dynamic> decodeMessages(dynamic messages) {
    return JSON.decode(messages);
  }
}
```

Once you've implemented your custom MessageBus in either TypeScript or Dart you can tell angular to use it like 
so:
In TypeScript:
```TypeScript
// index.ts, running on the UI side
import {bootstrapUICommon} from 'angular2/src/web_workers/ui/impl';
var bus = new MyAwesomeMessageBus();
bootstrapUICommon(bus);
```
```TypeScript
// background_index.ts, running on the application side
import {bootstrapWebWorkerCommon} from 'angular2/src/web_workers/worker/application_common';
import {MyApp} from './app';
var bus = new MyAwesomeMessageBus();
bootstrapWebWorkerCommon(MyApp, bus);
```
In Dart:
```Dart
// index.dart, running on the UI side
import 'package:angular2/src/web_workers/ui/impl.dart' show bootstrapUICommon;
import "package:angular2/src/core/reflection/reflection.dart";
import "package:angular2/src/core/reflection/reflection_capabilities.dart";

main() {
  reflector.reflectionCapabilities = new ReflectionCapabilities();
  var bus = new MyAwesomeMessageBus();
  bootstrapUiCommon(bus);
}
```
```Dart
// background_index.dart, running on the application side
import "package:angular2/src/web_workers/worker/application_common.dart" show bootstrapWebWorkerCommon;
import "package:angular2/src/core/reflection/reflection.dart";
import "package:angular2/src/core/reflection/reflection_capabilities.dart";
import "./app.dart" show MyApp;

main() {
    reflector.reflectionCapabilities = new ReflectionCapabilities();
    var bus = new MyAwesomeMessageBus();
    bootstrapWebWorkerCommon(MyApp, bus);
}
```
Notice how we call `bootstrapUICommon` instead of `bootstrap` from the UI side. `bootstrap` spans a new WebWorker
/ Isolate and attaches the default angular MessageBus to it. If you're using a custom MessageBus you are 
responsible for setting up the application side and initiating communication with it. `bootstrapUICommon` assumes
that the given MessageBus is already set up and can communicate with the application.
Similarly, we call `bootstrapWebWorkerCommon` instead of `boostrapWebWorker` from the application side. This is
because `bootstrapWebWorker` assumes you're using the default angular MessageBus and initializes a new one for you.

#### MessageBroker
The MessageBroker is a higher level messaging abstraction that sits on top of the MessageBus. It is used when you
want to execute code on the other side of a runtime boundary and may want to receive the result.
There are two types of MessageBrokers. The `ServiceMessageBroker` is used by the side that actually performs
an operation and may return a result. Conversely, the `ClientMessageBroker` is used by the side that requests that
an operation be performed and may want to receive the result.

##### Using the MessageBroker In Your Application
To use MessageBrokers in your application you must initialize both a `ClientMessageBroker` and a
`ServiceMessageBroker` on the same channel. You can then register methods with the `ServiceMessageBroker` and
instruct the `ClientMessageBroker` to run those methods. Below is a lightweight example of using
MessageBrokers in an application. For a more complete example, check out the `WebWorkerRenderer` and
`MessageBasedRenderer` inside the Angular WebWorker code.

###### Using the MessageBroker in TypeScript
```TypeScript
// index.ts, which is running on the UI with a method that we want to expose to a WebWorker
import {bootstrap} from 'angular2/web_worker/ui';

var instance = bootstrap("loader.js");
var broker = instance.app.createServiceMessageBroker("My Broker Channel");

// assume we have some function doCoolThings that takes a string argument and returns a Promise<string>
broker.registerMethod("awesomeMethod", [PRIMITIVE], (arg1: string) => doCoolThing(arg1), PRIMITIVE);
```
```TypeScript
// background.ts, which is running on a WebWorker and wants to execute a method on the UI
import {Component, View, ClientMessageBrokerFactory, PRIMITIVE, UiArguments, FnArgs}
from 'angular2/web_worker/worker';

@Component(...)
@View(...)
export class MyComponent {
  constructor(brokerFactory: ClientMessageBrokerFactory) {
    var broker = brokerFactory.createMessageBroker("My Broker Channel");
    
    var arguments = [new FnArg(value, PRIMITIVE)];
    var methodInfo = new UiArguments("awesomeMethod", arguments);
    broker.runOnService(methodInfo, PRIMTIVE).then((result: string) => {
      // result will be equal to the return value of doCoolThing(value) that ran on the UI.
    });
  }
}
```
###### Using the MessageBroker in Dart
```Dart
// index.dart, which is running on the UI with a method that we want to expose to a WebWorker
import 'package:angular2/web_worker/ui.dart';

main() {
  var instance = bootstrap("background.dart");
  var broker = instance.app.createServiceMessageBroker("My Broker Channel");
  
  // assume we have some function doCoolThings that takes a String argument and returns a Future<String>
  broker.registerMethod("awesomeMethod", [PRIMITIVE], (String arg1) => doCoolThing(arg1), PRIMITIVE);
}

```
```Dart
// background.dart, which is running on a WebWorker and wants to execute a method on the UI
import 'package:angular2/web_worker/worker.dart';

@Component(...)
@View(...)
class MyComponent {
  MyComponent(ClientMessageBrokerFactory brokerFactory) {
    var broker = brokerFactory.createMessageBroker("My Broker Channel");
    
    var arguments = [new FnArg(value, PRIMITIVE)];
    var methodInfo = new UiArguments("awesomeMethod", arguments);
    broker.runOnService(methodInfo, PRIMTIVE).then((String result) {
      // result will be equal to the return value of doCoolThing(value) that ran on the UI.
    });
  }
}
```
Both the client and the service create new MessageBrokers and attach them to the same channel.
The service then calls `registerMethod` to register the method that it wants to listen to. Register method takes
four arguments. The first is the name of the method, the second is the Types of that method's parameters, the
third is the method itself, and the fourth (which is optional) is the return Type of that method.
The MessageBroker handles serializing / deserializing your parameters and return types using angular's serializer.
However, at the moment the serializer only knows how to serialize angular classes like those used by the Renderer.
If you're passing anything other than those types around in your application you can handle serialization yourself
and then use the `PRIMITIVE` type to tell the MessageBroker to avoid serializing your data.

The last thing that happens is that the client calls `runOnService` with the name of the method it wants to run, 
a list of that method's arguments and their types, and (optionally) the expected return type.
