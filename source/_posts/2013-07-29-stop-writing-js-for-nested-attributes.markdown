---
layout: post
title: Stop writing JS for nested attributes
date: 2013-07-29 20:41
comments: true
categories:
---

I've been re-writing/copying the same JS to deal with adding/removing nested
attributes for far too long. On a project I recently started, I was fortunate
enough to stumble across
<a href="https://github.com/patbenatar/jquery-nested_attributes">
patbenatar/jquery-nested_attributes</a>. This jquery plugin makes handling
nested objects a cinch.

The simplest usage looks like this:

```coffeescript
$("#container").nestedAttributes(->
  bindAddTo: $("#add_another")
)
```

In the above example, `#container` refers to a DOM element whose immediate
descendants are considered to be sets of nested attributes. When the user
clicks the link referenced by `#add_another` the plugin automatically clones a
set of fields and appends to the DOM.

In case you need a bit more flexibility, there are a slew of options available:

```js
{
  collectionName: false,         // If not provided, we will attempt to autodetect. Provide this for complex collection names
  bindAddTo: false,              // Required unless you are implementing your own add handler (see API below). The single DOM element that when clicked will add another set of fields
  removeOnLoadIf: false,         // Function. It will be called for each existing item, return true to remove that item
  collectIdAttributes: true,     // Attempt to collect Rail's ID attributes
  beforeAdd: false,              // Function. Callback before adding an item
  afterAdd: false,               // Function. Callback after adding an item
  beforeMove: false,             // Function. Callback before updating indexes on an item
  afterMove: false,              // Function. Callback after updating indexes on an item
  beforeDestroy: false,          // Function. Callback before destroying an item
  afterDestroy: false,           // Function. Callback after destroying an item
  destroySelector: '.destroy',   // Pass in a custom selector of an element in each item that will destroy that item when clicked
  deepClone: true,               // Do you want jQuery to deep clone the element? Deep clones preserve events. Undesirable when using BackBone views for each element.
  $clone: null                   // Pass in a clean element to be used when adding new items. Useful when using plugins like jQuery UI Datepicker or Select2. Use in conjunction with `afterAdd`.
}
```

In my case, I am using select2 to provide some fancy select elements. The only
problem was select2 wasn't playing nice--I couldn't unbind select2 from the
select element and as a result pressing the "add another" link made the form
unusable. After a quick iteration with the plugin's author on github, I learned
about the `$clone` option. You can use `$clone` to pass in a "clean" element
that will get appended to the DOM when the "add another" link is pressed.

To take advantage of the `$clone` option you just have to get a copy of your
DOM element before binding any other JS to it:

```coffeescript
clone = $('.nested-object-fields:first').clone()

$(".container").nestedAttributes(
  bindAddTo: $(".add-another")
  $clone: clone
  afterAdd: (el) ->
  ...
)
```

Deleting items is just as easy. I got tripped up by having a hidden field for
`_destroy`. You don't need it! You just need to have an element with
`"destroy"` as the class. When this element is clicked the plugin automatically
adds the `_destroy` hidden field for you.

I can't imagine an easier way to manage nested attributes. Thanks, @patbenatar,
for an awesome plugin.
