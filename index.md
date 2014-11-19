# Angular 2.0

## index

* [about](#about)
* [atscript](#atscript)
* [Dependency Injection](#dependency-injection)
  * [Annotations](#annotations)
  * [Instance-scope](#instance-scope)
  * [Child-injectors](#child-injectors)
* [Templating & Databinding](#templating-and-databinding)
  * [Dynamic Loading](#dynamic-loading)
  * [Directives](#directives)
  * [Controllers](#controllers)
  * [Template Syntax](#template-syntax)
* [Router](#router)

## about

* performance
* future of databinding
    web components require most frameworks NG included to adapt their databinding  
    not a specific amount of elements with well known events but random any component...

## atscript

* superset of ES6 spec
* type syntax of typescript
* extending the language with annotations

example

``` js
import {Component} from 'angular';  
import {Server} from './server';

@Component({selector: 'foo'})
export class MyComponent {  
  constructor(server:Server) {
      this.server = server;
  }
}
```

*`import` and `class` syntax are pure ES6*
*The constructor argument of type `Server` is typescript'ish and is used for assertion at runtime*
*`@Component` is an example of metadata annotations*

this transpiles into regular ES6 as:

``` js
import * as rtts from 'rtts';  
import {Component} from 'angular';  
import {Server} from './server';

export class MyComponent {  
  constructor(server) {
      rtts.types(server, Server);
      this.server = server;
  }
}

MyComponent.parameters = [{is:Server}];  
MyComponent.annotate = [  
  new Component({selector: 'foo'})
];
```

rtts = RunTime Type System ... a small library for type checking.

**Angular 2.0 is being written with AtScript, but that doesn't mean you have to write your application code with AtScript or know anything about AtScript to use Angular 2.0. You can easily write with TypeScript, ES6, ES5, CoffeeScript...whatever you like. At present, you'll get the best Angular experience by leveraging AtScript, due to its ability to automatically generate the metadata from language primitives, but it all translates to simple ES5 in the end**

## Dependency Injection

### Annotations

* Old DI was flawed in relation with minification
* Compared to the bigger DI frameworks in other Serverside Environements (Java, .NET), it lacks in several areas for example lifetime//scope control and child injectors.

``` js
// ES5 transpiled code

// DI
MyComponent.parameters = [{is:Server}];

// Inject via Annotations overrides DI via parameters syntax:
MyComponent.annotate = [new Inject(Server)];
MyComponent.annotate = [new Inject('my-string-token')];
```

*as long as the DI framework knows a module or object named `my-string-token` it can be used to inject*

### Instance Scope

In Angular 1.x each instance of a DI container were singletons. In Ng 2.0 this is also the default.  
It however required Services, Providers ... to get different behavior.
In Ng 2.0 the new DI has more powerful features **Scope Control**.  
Just ask for it if you want a new instance with every inject.

``` js
@TransientScope
export class MyClass { ... }
```
*also possible to create your own scope identifiers (and use with child injectors*

### Child Injectors

A child injector inherits from all of its parent's services with ability to override at child level.  
Example:  

    The new router has a "Child Routers" capability. 
    Internally, each child router creates its own child injector. 
    This allows for each part of the route to inherit services 
    from parent routes or to override those services during 
    different navigation scenarios.

**custom scopes and child injection is however considered more intermediate to advanced usage of the injector. Mostly used internal in Angular, but available for people wanting to do some heavy lifting...**

## Templating and Databinding

### Dynamic Loading

in angular 1.x it was very dificult, to dynamically load a fragment / directive,  
angular 2.0 built this system from scratch with async in mind.

### Directives

Angular 2.0, made directives simpler using:

* modules
* classes
* annotations
* ...

they divide them in 3 types:

* **component directievs** custom component (view + controller) if needed a router can map a route to a component.
* **decorator directives** decorates existing html with extra behavior example `ng-show`
* **template directives** transform html into reusable template, example `ng-if` `ng-repeat`
 
Contrary to what many doomsayers say, controllers are not dead in ng2.0. a Controller is 1 of the parts of a component.  
However rather than having to explicitly state a function as `Controller` now you can just use a regular class with annotations behind your markup.

An example of a tabContainer (partial)

``` js
@ComponentDirective({
    selector:'tab-container',
    directives:[NgRepeat]
})
export class TabContainer {  
    constructor(panes:Query<Pane>) {
        this.panes = panes;
    }

    select(selectedPane:Pane) { ... }
}
```
features:

* controller is just a regular class, its constructor has dependencies injected. Using child injectors it gets a query injected. a Query is a special collection to keep the tabContainer in sync with its child panes. It knows when items are added / removed / ...
* the ComponentDirective annotation identifies this class as a component. the selector is a css selector used to match on a html element. Any element with this class will be turned into a tabContainer.
* the ComponentDirective annotation also specifies directive dependencies in the template. this Component has a dependency on NgRepeat.
* binding shit on this class `this.myvar = 2` is immediately accessible in the template. (see this class as the scope for your component)

example of a decorator directive

``` js
@DecoratorDirective({
    selector:'[ng-show]',
    bind: { 'ngShow': 'ngShow' },
    observe: {'ngShow': 'ngShowChanged'}
})
export class NgShow {  
    constructor(element:Element) {
        this.element = element;
    }

    ngShowChanged(newValue){
        if(newValue){
            this.element.style.display = 'block';
        }else{
            this.element.style.display = 'none';
        }
    }
}
```

features:
* again a class with annotations
* it binds to any item with the selector ng-show (an attribute with that name)
* the bind metadata is required if you want your property to be bindable in HTML
* the observe metadata is required if you want to be notified of changes

this is pure hypothetical, not the real NgShow implementation, just a demo

now, a template directive

``` js
@TemplateDirective({
    selector: '[ng-if]',
    bind: {'ngIf': 'ngIf'},
    observe: {'ngIf': 'ngIfChanged'}
})
export class NgIf {  
    constructor(viewFactory:BoundViewFactory, viewPort:ViewPort) {
        this.viewFactory = viewFactory;
        this.viewPort = viewPort;
        this.view = null;
    }

    ngIfChanged(value) {
        if (!value && this.view) {
            this.view.remove();
            this.view = null;
        }

        if (value) {
            this.view = this.viewFactory.createView();
            this.view.appendTo(this.viewPort);
        }
    }
}
```

features: 

* again class with annotations, and DI
* also binds to attributes, and listens for changes
* a viewport represents a place in the DOM for your component
* the createfactory instantiates a template with some data and is appended here to the viewport.

### Controllers

In Angular 1.x Directives and Controllers were two different things. There were different APIs and different capabilities. 
In Angular 2.0, since they removed the DDO and made Directives class-based, we were able to unify Directives and Controllers 
into the Component model. 

Now when defining routes, map a route to a ComponentDirective.

example:

``` js
@ComponentDirective
export class CustomerEditController {  
    constructor(server:Server) {
        this.server = server;
        this.customer = null;
    }

    activate(customerId) {
        return this.server.loadCustomer(customerId)
            .then(response => this.customer = response.customer);
    }
}
```

### Template Syntax

Example template of the hypothetical tabcontainer we saw before:

``` html
<template>  
    <div class="border">
        <div class="tabs">
            <div [ng-repeat|pane]="panes" class="tab" (^click)="select(pane)">
                <img [src]="pane.icon"><span>${pane.name}</span>
            </div>
        </div>
        <content></content>
    </div>
</template>  
```

* Ugly but still **valid html** according to the spec
* WIP, and not final

features: 

* attribute name surrounded with `[]`: tells us the value of the attribute, has a binding expression.
* expression surrounded with `${}`: tells you that there's an expression that should be interpolated into the content as a string.
* both bindings are unidirectional from controller to view.
* ng-repeat surrounded with `[]` is an expression but also has `|pane` it creates a new local var `pane`.
* parantheses `()`: tell us we attach an expression to an event handler.
* a `^` within parantheses tell us we don't bind to the element but let it bubble and be handled at document level.

one of the reasons for the current binding syntax:

* hard to distinguish event from parameter `<div foo="x" bar="y"...`

**This area has been heavily debated for months by the team. The above syntax is not unanimously agreed upon but it was a recent majority decision. There are lots of considerations when working out a binding syntax. [more info](https://docs.google.com/document/d/1kpuR512G1b0D8egl9245OHaG0cFh0ST0ekhD_g8sxtI/edit?usp=sharing)**

## Router

* child routing (no more 1 router to rule them all)
* screen activation (take charge example, user want's to exit but data is not saved yet...)
  * canActivate / activate
  * canDeactivate / deactivate
* Design - pluggable, can add extra 'middleware' as it were,... (ex. authentication)

---

remember, it's all WIP

---


# DISCUSS!
