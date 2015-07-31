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
  * [String Interpolation](#string-interpolation)
  * [Inline Templates](#inline-templates)
  * [Template Microsyntax](#template-microsyntax)
* [Change detection](#change-detection)
  * [Immutable Objects](#immutable-objects) 
  * [Observable Objects](#observable-objects)
* [Core](#core)
* [DI](#di)
  * [Core Abstractions](#core-abstractions)
  * [Key and Token](#key-and-token)
  * [Injector](#injector)
  * [Bindings](#bindings)
  * [Dependencies](#dependencies)
  * [Async](#async)
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
### Core
-----------------------------------
<!-- -------------------------- -->
### DI
-----------------------------------

#### Core Abstractions

The library is built on top of the following core abstractions: `Injector`, `Binding`, and `Dependency`.

* An injector resolves dependencies and creates objects.
* A binding maps a token to a factory function and a list of dependencies. So a binding defines how to create an object. A binding can be synchronous or asynchronous.
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



#### Key and Token

Any object can be a token. For performance reasons, however, DI does not deal with tokens directly, and, instead, wraps every token into a Key. See the section on "Key" to learn more.



##### Example

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

In this example we create two bindings: one for Car and one for Engine. `@Inject(Engine)` declares that Car depends on Engine.



#### Injector

An injector instantiates objects lazily, only when needed, and then caches them.

Compare

``` javascript
var car = inj.get(Car); //instantiates both an Engine and a Car
```

with

``` javascript
var engine = inj.get(Engine); //instantiates an Engine
var car = inj.get(Car); //instantiates a Car
```

and with

``` javascript
var car = inj.get(Car); //instantiates both an Engine and a Car
var engine = inj.get(Engine); //reads the Engine from the cache
```

To avoid bugs make sure the registered objects have side-effect-free constructors. If it is the case, an injector acts like a hash map with all of the registered objects created at once.


##### Child Injector

Injectors are hierarchical.

``` javascript
var child = injector.resolveAndCreateChild([
	bind(Engine).toClass(TurboEngine)
]);

var car = child.get(Car); // uses the Car binding from the parent injector and Engine from the child injector
```


#### Bindings

You can bind to a class, a value, or a factory. It is also possible to alias existing bindings.

``` javascript
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

``` javascript
var inj = Injector.resolveAndCreate([
	bind(Car).toFactory((e) => new Car(), ["engine!"]),
	bind("engine!").toClass(Engine)
]);
```

If you want to alias an existing binding, you can do so using `toAlias`:

``` javascript
var inj = Injector.resolveAndCreate([
	bind(Engine).toClass(Engine),
	bind("engine!").toAlias(Engine)
]);
```
which implies `inj.get(Engine) === inj.get("engine!")`.

Note that tokens and factory functions are decoupled.

``` javascript
bind("some token").toFactory(someFactory);
```

The `someFactory` function does not have to know that it creates an object for `some token`.


##### Default Bindings

Injector can create binding on the fly if we enable default bindings.

``` javascript
var inj = Injector.resolveAndCreate([], {defaultBindings: true});
var car = inj.get(Car); //this works as if `bind(Car).toClass(Car)` and `bind(Engine).toClass(Engine)` were present.
```

This can be useful in tests, but highly discouraged in production.


#### Dependencies

A dependency can be synchronous, asynchronous, or lazy.

``` javascript
class Car {
	constructor(@Inject(Engine) engine) {} // sync
}

class Car {
	constructor(engine:Engine) {} // syntax sugar for `constructor(@Inject(Engine) engine:Engine)`
}

class Car {
	constructor(@InjectPromise(Engine) engine:Promise) {} //async
}

class Car {
	constructor(@InjectLazy(Engine) engineFactory:Function) {} //lazy
}
```

* The type annotation is used by DI only when no @Inject annotations are present.
* `InjectPromise` tells DI to inject a promise (see the section on async for more information).
* `InjectLazy` enables deferring the instantiation of a dependency by injecting a factory function.



#### Async

Asynchronicity makes code hard to understand and unit test. DI provides two mechanisms to help with it: asynchronous bindings and asynchronous dependencies.

Suppose we have an object that requires some data from the server.

This is one way to implement it:

``` javascript
class UserList {
	loadUsers() {
		this.usersLoaded = fetchUsersUsingHttp();
		this.usersLoaded.then((users) => this.users = users);
	}
}

class UserController {
	constructor(ul:UserList){
		this.ul.usersLoaded.then((_) => someLogic(ul.users));
	}
}
```

Both the UserList and UserController classes have to deal with asynchronicity. This is not ideal. UserList should only be responsible for dealing with the list of users (e.g., filtering). And UserController should make ui-related decisions based on the list. Neither should be aware of the fact that the list of users comes from the server. In addition, it clutters unit tests with dummy promises that we are forced to provide.

The DI library supports asynchronous bindings, which can be used to clean up UserList and UserController.

``` javascript
class UserList {
	constructor(users:List){
		this.users = users;
	}
}

class UserController {
	constructor(ul:UserList){
	}
}

var inj = Injector.resolveAndCreate([
	bind(UserList).toAsyncFactory(() => fetchUsersUsingHttp().then((u) => new UserList(u))),
	UserController
])

var uc:Promise = inj.asyncGet(UserController);
```

Both UserList, UserController are now async-free. As a result, they are easy to reason about and unit test. We pushed the async code to the edge of our system, where the initialization happens. The initialization code tends to be declarative and relatively simple. And it should be tested with integration tests, not unit tests.

Note that asynchronicity have not disappeared. We just pushed out it of services.

DI also supports asynchronous dependencies, so we can make some of our services responsible for dealing with async.

``` javascript
class UserList {
	constructor(users:List){
		this.users = users;
	}
}

class UserController {
	constructor(@InjectPromise(UserList) ul:Promise){
	}
}

var inj = Injector.resolveAndCreate([
	bind(UserList).toAsyncFactory(() => fetchUsersUsingHttp().then((u) => new UserList(u))),
	UserController
])

var uc = inj.get(UserController);
```

We can get an instance of UserController synchronously. It is possible because we made UserController responsible for dealing with asynchronicity, so the initialization code does not have to.

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
