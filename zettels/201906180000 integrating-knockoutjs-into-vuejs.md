---
title: "How to nest Knockout components in a Vue app"
slug: "integrating-knockoutjs-into-vuejs"
date: "2019-06-18"
summary: ""
---

#software-development #web-development

When I first got my feet wet developing web applications about five years ago I quickly figured I needed to go the SPA way as that was the way I could implement all the futuristic visions we had at the time. Now several years later the JS ecosystem has evolved further, and while the technology we used back in the days still works fine, there are many reasons to switch to a newer platform. Shiny features everywhere!

With this background I have gotten on a long due adventure to figure out whether it was possible to use [Knockout](https://knockoutjs.com/) components within a Vue application. The idea is that we could rewrite the shell using [Vue](https://vuejs.org/) in order to get much of these new shiny features, while still reusing the existing components in order to prevent having to rebuild the whole app. Note that this 'shell' was previously provided through the use of [Durandal](http://durandaljs.com/).

Knockout is in itself - thankfully - a library which can be used wherever you have javascript and a DOM. The next thing you would want to make sure is that the components and their dependencies can easily be moved to your new Vue application.

## Getting Knockout components to render within Vue

Our goal is to be able to render a Knockout view and model within a wrapper. This wrapepr is a Vue component which accepts an argument which contains the view/model. In my case this combination was mostly coupled together.

We'll call this wrapper the `KnockoutFrame`. This component has the following definition:

```js
<template>
  <span></span>
</template>

<script>
import ko from "knockout";

export default {
  props: ["component"],
  mounted() {
    this.$el.innerHTML = this.component.template;
    ko.applyBindings(this.component, this.$el);
  }
};
</script>
```

One of the important things to keep in the back of your head is that Vue uses a virtual DOM, while Knockout does not. The direct result of this is that it's fairly complex to mix usage of Knockout and Vue, but that's not our goal anyway.


## Moving a Knockout component

Now that we have a wrapper we still need something to display on the page. This is where our migration step comes in place. Due to the complexity of front-end development it would be a fairly complex task to write a 'one size fits it all' solution, and the approach showed in this post is just a suggestion of what you could do.

The core idea is that we embed a view and model within a javascript component. While this can be achieved via multiple approaches the most straightforward one is to create a new class containing a template and model, which then can be instantiated whenever you embed it into a `KnockoutFrame`, which makes it clear when a new instance is used.

```js
export class Component {
  template = ``
  model = { }
}

export default new Component
```

### An example

An example of a Knockout component working in Vue can be grabbed from the Knockout website. All examples given there work well with just a slight modification:

```js
import ko from 'knockout'

export class AppViewModel {
  constructor() {
    this.firstName = ko.observable('Planet');
    this.lastName = ko.observable('Earth');
  
    this.fullName = ko.pureComputed({
      read: function () {
        return this.firstName() + " " + this.lastName();
      },
      write: function (value) {
        var lastSpacePos = value.lastIndexOf(" ");
        if (lastSpacePos > 0) { // Ignore values with no space character
          this.firstName(value.substring(0, lastSpacePos)); // Update "firstName"
          this.lastName(value.substring(lastSpacePos + 1)); // Update "lastName"
        }
      },
      owner: this
    });
  }

  template =
    `<div>First name: <span data-bind="text: firstName"></span></div>
    <div>Last name: <span data-bind="text: lastName"></span></div>
    <div class="heading">Hello, <input data-bind="textInput: fullName"/></div>`
}
```

## Javascript classes

It is important to realise that different ways of importing this component will result in different behaviour.

The following approach will result in a singleton instance of `Component`:

```js
import Component from './Component'

console.log(Component)
// Component { template: ``, model: { } }
```

While this approach will result in having an function which can then be used for instantiation:

```js
import { Component } from './Component'

console.log(Component)
// [Function: Component]

console.log(new Component())
// Component { template: ``, model: { } }
```

I strongly recommend anyone going this way to read themselves into the ES6 syntax and making sure it is correctly understood before starting this yourney. 


## What's next?

There are many approaches to iterate on this idea. For me personally however, this is just a stepping stone to be able to start working with Vue quickly, without the hassle of rewriting a whole application at once. However, be aware for legacy code. This is one of these things that one would rather not touch after it's working fine, and special care and planning should be taken when planning to use this approach to push an application forward.
