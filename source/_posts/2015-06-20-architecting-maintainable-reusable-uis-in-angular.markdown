---
layout: post
title: "Architecting Maintainable, Reusable UIs in Angular: A Case Study"
date: 2015-06-20 13:01
comments: true
categories: [javascript, angular, architecture, frontend]
published: true
---

After spending substantial time working in-depth in Angular land, and with further influence from incursions into Flux &amp; React, I've come to develop certain opinions on how to best architect non-trivial, data-driven UI flows in an Angular application.

What follows is a case study of a real-world UI problem, solved with the guidance of well-established principles and patterns in software design.

[Read the code on Github](https://github.com/kvcrawford/ng-permutation-builder) &bull; [See the live demo](http://kvcrawford.github.io/ng-permutation-builder/)

## The User Story
Jill works for *The Widget Factory*, a company in the business of making widgets. Oftentimes, she wants to be able to test how slightly different widgets perform against each other.

Rather than waste time creating the otherwise-same widget several times over, she would like to be able to quickly generate the different permutations, and be done with it.

<!-- more -->

## Some Guiding Principles
Before we begin, I'd like to highlight some design principles that will guide our implementation. While the purpose of this article is to *demonstrate* rather than explain these topics in detail, I've included links for further reading.

* The Single Responsibility Principle (SRP) ^[1](http://en.wikipedia.org/wiki/Single_responsibility_principle) ^[2](http://www.objectmentor.com/resources/articles/srp.pdf)
* Separation of Concerns (SoC) ^[3](http://en.wikipedia.org/wiki/Separation_of_concerns)
* Don't Repeat Yourself (DRY) ^[4](http://en.wikipedia.org/wiki/Don%27t_repeat_yourself)

We'll proceed iteratively. First, make it work—then, refactor.

## First: how do we generate permutations?
TDD for this type of stuff is a must. Consider this object of *permutable attributes*:

```coffeescript
permutable_attributes =
  name: ['Foobar', 'Bizbat']
  description: ['I pity the foo.', 'Lorem ipsum.']
```

From this, we would generate 4 possible permutations:

```coffeescript
[
  name: 'Foobar'
  description: 'I pity the foo.'
,
  name: 'Foobar'
  description: 'Lorem ipsum.'
,
  name: 'Bizbat'
  description: 'I pity the foo.'
,
  name: 'Bizbat'
  description: 'Lorem ipsum.'
]
```

Let's expand on those expectations: [permutation-factory.spec.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/test/permutation-factory.spec.coffee)

```coffeescript
describe 'permutationFactory:', ->
  permutationFactory = null

  beforeEach module 'app.permutation'

  beforeEach inject (
    _permutationFactory_
  ) ->
    permutationFactory = _permutationFactory_

  describe 'permute:', ->
    # Expected result for 2*2
    result_2x2 = [
      name: 'foo'
      attr: 'biz'
    ,
      name: 'foo'
      attr: 'bat'
    ,
      name: 'bar'
      attr: 'biz'
    ,
      name: 'bar'
      attr: 'bat'
    ]

    it 'Should generate 4 permutations from 2*2 attributes.', ->
      permutable_attributes =
        name: ['foo', 'bar']
        attr: ['biz', 'bat']

      permutations = permutationFactory.permute permutable_attributes

      expect(permutations).toEqual result_2x2

    it 'Should ignore attributes without any values.', ->
      # Some attributes won't be required--we want to skip those.

      permutable_attributes =
        name: ['foo', 'bar']
        empty: []
        attr: ['biz', 'bat']

      permutations = permutationFactory.permute permutable_attributes

      expect(permutations).toEqual result_2x2

    it 'Should ignore empty attributes at the end of the object.', ->
      permutable_attributes =
        name: ['foo', 'bar']
        attr: ['biz', 'bat']
        empty: []

      permutations = permutationFactory.permute permutable_attributes

      expect(permutations).toEqual result_2x2

    it 'Should generate 6 permutations from 2*1*0*3 attributes.', ->
      permutable_attributes =
        name: ['foo', 'bar']
        type: ['biz']
        empty: []
        desc: ['bing', 'bang', 'boom']

      permutations = permutationFactory.permute permutable_attributes

      expect(permutations).toEqual [
        name: 'foo'
        type: 'biz'
        desc: 'bing'
      ,
        name: 'foo'
        type: 'biz'
        desc: 'bang'
      ,
        name: 'foo'
        type: 'biz'
        desc: 'boom'
      ,
        name: 'bar'
        type: 'biz'
        desc: 'bing'
      ,
        name: 'bar'
        type: 'biz'
        desc: 'bang'
      ,
        name: 'bar'
        type: 'biz'
        desc: 'boom'
      ]

    it 'Should invoke an optional callback for each permutation', ->
      callback = jasmine.createSpy 'callback'

      permutable_attributes =
        name: ['foo', 'bar']
        attr: ['biz', 'bat']

      permutations = permutationFactory.permute permutable_attributes, callback

      expect(callback.calls.count()).toBe 4

      for i in [0..3]
        expect(callback.calls.argsFor(i)).toEqual [result_2x2[i]]

```

With tests in place, *now* we can do our implementation: [permutation-factory.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/service/permutation-factory.coffee)

```coffeescript
###
@name permutationFactory
@description
Utility service for generating permutations of a resource.
###

angular.module 'app.permutation'
.factory 'permutationFactory', ->

  ###
  @name permute
  @description
  Generates permutations from a permutable_attributes object.

  @param {Object} permutable_attributes
  @param {[Callback]} callback - Optional, invoked for each permutation

  @callback callback
  @param {Object} permutation

  @returns {Array} - Collection of permutations

  @example
  ```coffeescript
  permutations = permutationFactory.permute
    name: ['Foobar', 'Bizbat']
    description: ['I pity the foo.', 'Lorem ipsum.']
  ```
  ###

  permute: (permutable_attributes, callback) ->
    permutations = []

    recurse = (keys, payload = {}, position = 0) ->
      # We've finished constructing the permutation, exit call stack
      if position is keys.length
        permutation = _.clone payload
        callback? permutation
        permutations.push permutation

        return

      # Grab the current key
      key = keys[position]

      # There are no values for this attribute, skip it
      if permutable_attributes[key].length is 0
        recurse keys, payload, position + 1

      # Otherwise, recurse for each possible value of this attribute
      else
        for value in permutable_attributes[key]
          payload[key] = value

          recurse keys, payload, position + 1

    recurse Object.keys permutable_attributes

    return permutations

```

With our `permute` algorithm complete, now we can wire up a UI.

## A First Iteration
One of the most common pitfalls seen in Angular apps are bloated controllers. It can be tempting to wedge bits of logic here and there, as it's easy at the time. Unfortunately, the controller quickly turns into a tangled mess. Consider the following implementation of our permutation builder:

[spaghetti-widget-builder-controller.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/widget/controller/spaghetti-widget-builder-controller.coffee)

```coffeescript
angular.module 'app.widget'
.controller 'SpaghettiWidgetBuilderController', (
  $scope
  $state
  permutationFactory
  widgetFactory
  widgetStore
) ->
  # Placeholder object to hold references for our ngForm instances
  $scope.forms = {}

  ###
  @name initialize
  @description
  Resets state, called if the user hits the reset button.

  The `do` immediately invokes our method to initialize state when
  controller first loads.
  ###

  do $scope.initialize = ->
    $scope.permutations = []

    $scope.permutable_attributes =
      name: []
      description: []

    # For binding to the form with ngModel
    $scope.attributes =
      name: ''
      description: ''

  # We need to specify which fields are required. We can't just use a
  # 'required' attribute on the input tag, because it will no longer
  # be required once at least 1 value has been entered.
  $scope.required =
    name: true
    description: false

  ###
  @name buildPermutations
  @description
  Private function called every time an attribute is added or removed.
  ###

  buildPermutations = ->
    $scope.permutations.length = 0

    # At least we have the `permutationFactory` and `widgetFactory`,
    # which are more obvious as candidates for separate services.
    permutationFactory.permute $scope.permutable_attributes, (permutation) ->
      if widgetFactory.validate permutation
        $scope.permutations.push permutation

  ###
  @name createPermutations
  @description
  Persists the permutations and redirects to home page.
  ###

  $scope.createPermutations = ->
    widgetStore.addWidgets $scope.permutations

    $state.go 'home'

  ###
  @name addAttribute
  @description
  Adds an attribute to generate permutations from, then empties the form input.

  @param {String} key - Attribute name
  ###

  $scope.addAttribute = (key) ->
    # Tokenize the value
    $scope.permutable_attributes[key].push $scope.attributes[key]

    # Then empty the form input
    $scope.attributes[key] = ''

    buildPermutations()

  ###
  @name isRequired
  @description
  Used with ng-required, determines if at least 1 value has been entered or not.
  This view logic would fit much more nicely in a directive.

  @param {String} key - Attribute name

  @returns {Boolean}
  ###

  $scope.isRequired = (key) ->
    return (
      $scope.required[key] and
      $scope.permutable_attributes[key].length is 0
    )

  ###
  @name isDisabled
  @description
  We don't want the submit button to be enabled if input is empty.
  Also a good candidate for inclusion in a directive.

  @param {String} key - Attribute name

  @returns {Boolean}
  ###

  $scope.isDisabled = (key) ->
    return _.isEmpty $scope.attributes[key]

  ###
  @name removeAttribute
  @description
  Removes a permutable attribute, then re-generates permutations.

  @param {String} key - Attribute name
  @param {Integer} index - Index in the permutable attribute array.
  ###

  $scope.removeAttribute = (key, index) ->
    $scope.permutable_attributes[key].splice index, 1

    buildPermutations()

```

[spaghetti-widget-builder.jade](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/widget/spaghetti-widget-builder.jade)

{% codeblock lang:jade %}
{% raw %}
//- Can you spot all the repeating markup? Just imagine if we had more fields!
.container-fluid
  .row
    .col-md-12
      p.lead Let's build some widgets.

  .row
    .col-lg-6.col-md-8.col-sm-6
      form.form-group(
        ng-submit = "addAttribute('name')"
        name = "forms.name"
        ng-class = "{ 'has-error': forms.name.$invalid && forms.name.value.$touched }"
      )
        label.control-label(
          for = "widget_name"
        ) Name*

        .input-group
          input.form-control(
            id = "widget_name"
            name = "value"
            type = "text"
            ng-model = "attributes.name"
            ng-required = "isRequired('name')"
          )

          .input-group-btn
            button.btn.btn-default(
              type = "submit"
              ng-disabled = "isDisabled('name')"
            ) Add

      form.form-group(
        ng-submit = "addAttribute('description')"
        name = "forms.description"
        ng-class = "{ 'has-error': forms.description.$invalid && forms.description.value.$touched }"
      )
        label.control-label(
          for = "widget_description"
        ) Description

        .input-group
          input.form-control(
            id = "widget_description"
            name = "value"
            type = "text"
            ng-model = "attributes.description"
            ng-required = "isRequired('description')"
          )

          .input-group-btn
            button.btn.btn-default(
              type = "submit"
              ng-disabled = "isDisabled('description')"
            ) Add

    .col-lg-4.col-md-4.col-sm-6
      .panel.panel-default
        .panel-heading
            h4.panel-title {{permutations.length}} Widgets Built

        .panel-body
          .panel.panel-default
            .panel-heading
              strong Name ({{permutable_attributes.name.length}})

            ul.list-group
              li.list-group-item(
                ng-repeat = "name in permutable_attributes.name"
              )
                span {{name}}
                button.btn.close(
                  ng-click = "removeAttribute('name', $index)"
                ) &times;

          .panel.panel-default
            .panel-heading
              strong Description ({{permutable_attributes.description.length}})

            ul.list-group
              li.list-group-item(
                ng-repeat = "description in permutable_attributes.description"
              )
                span {{description}}
                button.btn.close(
                  ng-click = "removeAttribute('description', $index)"
                ) &times;

        .panel-footer
          .btn-group
            button.btn.btn-primary(
              type = "button"
              ng-click = "createPermutations()"
              ng-disabled = "permutations.length === 0"
            ) Submit

            button.btn.btn-default(
              type = "button"
              ng-click = "initialize()"
              ng-disabled = "permutations.length === 0"
            ) Reset
{% endraw %}
{% endcodeblock %}

It works! Cool! But, there are a few problems here:

* There's poor separation of concerns: view logic and state (`isRequired`, `isDisabled`) are intermingled with business logic and state.
* What if we want to add new features, like permutable images? Or videos? This controller and template will keep getting bigger.
* What if we want to add an additional step, to review the permutations we've generated before saving them? Having the data model so tightly coupled to the controller becomes problematic.
* What we have isn't very reusable. What if we want to create another permutation builder for *Gadgets*?

*Note that we still have the [permutationFactory](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/service/permutation-factory.coffee) and [widgetFactory](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/widget/service/widget-factory.coffee), which were more obvious candidates for separate services.*

## Teasing Out The Layers
By isolating our concerns into separate layers, we can create something that is both easier to maintain *and* reusable. There are two modes of thinking that I like to employ:

### 1. Think purely in terms of business logic and state
Without even considering a UI, how would you describe the state of our permutation builder? How would you design an API to manipulate that state?

Consider [Flux](https://facebook.github.io/flux/docs/overview.html#content)'s idea of a store:

> Stores contain the application state and logic. Their role is somewhat similar to a model in a traditional MVC, but they manage the state of many objects — they do not represent a single record of data like ORM models do. Nor are they the same as Backbone's collections. More than simply managing a collection of ORM-style objects, stores manage the application state for a particular domain within the application.

In an Angular app, we can implement something similar with a service, isolating the business logic and state for our permutation builder. This gives us a number of advantages:

* It's easier to test,
* It becomes easier to reuse and extend, and
* We avoid controller bloat, by properly isolating our concerns.

[permutation-builder-service.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/service/permutation-builder-service.coffee)

```coffeescript
###
@name PermutationBuilderService
@description
A base class that can be extended for use with different permutable resources.
###

angular.module 'app.permutation'
.factory 'PermutationBuilderService', (
  permutationFactory
) ->
  class PermutationBuilderService
    constructor: ->
      @initialize()

    ###
    @name initialize
    @description
    Method used to reset service to an empty state.
    Override this method to define attributes for a permutable resource.
    ###

    initialize: ->
      # Regular attributes that get added to each permutation
      @attributes = {}

      # Permutable attributes
      @permutable_attributes = {}

      # Collection of permutations
      @permutations = []

    ###
    @name addAttribute
    @description
    Adds a permutable attribute

    @param {String} key - Name of attribute
    @param {String|Number|Object|Array} value - Can be of any type

    @returns {Boolean} Whether attribute was added successfully or not.
    ###

    addAttribute: (key, value) ->
      bucket = @permutable_attributes[key]

      throw Error "Invalid key: '#{key}'" unless bucket?

      if _.contains bucket, value
        console.warn "'#{value}' already entered."

        return false

      # Add the value to permute against
      bucket.push value

      # And generate the permutations
      @buildPermutations()

      return true

    ###
    @name buildPermutations
    @description
    Builds permutations of the resource.

    @returns {Array} Collection of permutations
    ###

    buildPermutations: ->
      # Empty our collection of permutations from previous runs
      @permutations.length = 0

      permutationFactory.permute @permutable_attributes, (permutation) =>
        # Extend common attributes onto each permutation
        resource = _.extend permutation, @attributes

        @permutations.push resource

      return @permutations

    ###
    @name createPermutations
    @description
    Abstract method. Override to define how a permutable resource gets persisted.
    ###

    createPermutations: ->

    ###
    @name removeAttribute
    @description
    Removes a permutable attribute by index

    @param {String} key - Attribute name
    @param {Integer} index - Index of value in the permutable attribute array.

    @returns {Boolean} - true if item successfully removed
    ###

    removeAttribute: (key, index) ->
      bucket = @permutable_attributes[key]

      throw Error "Invalid key: '#{key}'" unless bucket?

      removed = bucket.splice index, 1

      @buildPermutations()

      return removed.length > 0

```

### 2. Think of your UI in terms of a tree of components
Look at your design. Look at your markup. Do you see any patterns? These parts of our UI are ripe for refactoring into directives.

Imagine being able to compose our view as such:

```jade
kc-permutation-builder(
  service = "PermutationBuilderService"
)
  .main
    kc-permutable-input(
      name = "name"
      type = "text"
      required
    ) Name

    kc-permutable-input(
      name = "description"
      type = "text"
    ) Description

  .sidebar
    kc-permutable-attribute(
      name = "name"
    ) Name

    kc-permutable-attribute(
      name = "description"
    ) Description

    button(
      type = "submit"
      ng-click = "submit()"
    ) Create Permutations
```

That's a lot more succinct, expressive, and reusable.

[permutation-builder-directive.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/directive/permutation-builder-directive.coffee)

```coffeescript
###
@name kcPermutationBuilder
@description
This serves as a way to bind an instance of a PermutationBuilderService and
expose its API to a group of `kcPermutableInput` directives.

Even though it has an isolate scope, it doesn't have any template, so it doesn't
introduce an isolate scope in the template in which its used.

@param {PermutationBuilderService} service - Or a subclass thereof
###

angular.module 'app.permutation'
.directive 'kcPermutationBuilder', ->
  restrict: 'E'
  controller: 'KcPermutationBuilderController'
  scope:
    service: '='

.controller 'KcPermutationBuilderController', (
  $scope
) ->
  @permutable_attributes = $scope.service.permutable_attributes

  @addAttribute =
    angular.bind $scope.service, $scope.service.addAttribute

  @removeAttribute =
    angular.bind $scope.service, $scope.service.removeAttribute

```

[permutable-input-directive.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/directive/permutable-input-directive.coffee)

```coffeescript
###
@name kcPermutableInput
@description
Encapsulates templating and view logic for a permutable input, which is its
own mini form. Makes for a flexible component that can be used to compose the
view for any type of permutable resource.

@param {String} name - Permutable attribute key

@example
`kc-permutable-input(name="title", type="text", required) Label`
###

angular.module 'app.permutation'
.directive 'kcPermutableInput', ->
  require: '^kcPermutationBuilder'
  restrict: 'E'
  templateUrl: '/permutation/_permutable-input.html'
  transclude: true

  scope:
    name: '@name'

  link: (scope, element, attrs, kcPermutationBuilder) ->
    # Ensure input IDs are unique
    scope.input_id = _.uniqueId 'permutatable_input_'

    scope.state =
      value: ''
      is_required: attrs.required?

    ###
    @name isDisabled
    @description
    We don't want the submit button to be enabled if input is empty.

    @returns {Boolean}
    ###

    scope.isDisabled = ->
      return _.isEmpty scope.state.value

    ###
    @name isRequired
    @description
    An input is no longer required if at least 1 value has already been entered.

    @returns {Boolean}
    ###

    scope.isRequired = ->
      return attrs.required? and
        kcPermutationBuilder.permutable_attributes[scope.name].length is 0

    ###
    @name submit
    @description
    Adds attribute and clears the input.
    ###

    scope.submit = ->
      is_added = kcPermutationBuilder.addAttribute(
        scope.name
        scope.state.value
      )

      if is_added
        scope.state.value = ''

```

[_permutable-input.jade](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/_permutable-input.jade)

{% codeblock lang:jade %}
{% raw %}
form.form-group(
  ng-submit = "submit()"
  name = "form"
  ng-class = "{ 'has-error': form.$invalid && form.value.$touched }"
)
  label.control-label(
    for = "{{input_id}}"
  )
    span(ng-transclude)
    span(ng-if="state.is_required") *

  .input-group
    input.form-control(
      id = "{{input_id}}"
      name = "value"
      type = "text"
      ng-model = "state.value"
      ng-required = "isRequired()"
    )

    .input-group-btn
      button.btn.btn-default(
        type = "submit"
        ng-disabled = "isDisabled()"
      ) Add

{% endraw %}
{% endcodeblock %}

[permutable-attribute-directive.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/directive/permutable-attribute-directive.coffee)

```coffeescript
###
@name kcPermutableAttribute
@description
Displays the values entered for a permutable attribute.
Allows user to remove a value.

@param {String} name - Permutable attribute key

@example
kc-permutable-attribute(
  name="description"
) Description
###

angular.module 'app.permutation'
.directive 'kcPermutableAttribute', ->
  require: '^kcPermutationBuilder'
  restrict: 'E'
  templateUrl: '/permutation/_permutable-attribute.html'
  transclude: true

  scope:
    name: '@name'

  link: (scope, element, attrs, kcPermutationBuilder) ->
    scope.permutable_attribute =
      kcPermutationBuilder.permutable_attributes[scope.name]

    ###
    @name removeAttribute
    @description
    Calls on service to remove attribute.

    @param {Integer} index - Value's index in the permutable attribute array.
    ###

    scope.removeAttribute = (index) ->
      kcPermutationBuilder.removeAttribute scope.name, index

```

[_permutable-attribute.jade](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/permutation/_permutable-attribute.jade)

{% codeblock lang:jade %}
{% raw %}
.panel.panel-default
  .panel-heading
    strong(ng-transclude)
    strong &nbsp;({{permutable_attribute.length}})

  ul.list-group
    li.list-group-item(
      ng-repeat = "attribute in permutable_attribute"
    )
      span {{attribute}}
      button.btn.close(
        ng-click = "removeAttribute($index)"
      ) &times;

{% endraw %}
{% endcodeblock %}

## The New and Improved Widget Builder
Now with the abstracted modules ready for use, look at how much leaner all of our widget-specific code is.

[widget-builder-service.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/widget/service/widget-builder-service.coffee)

```coffeescript
###
@name WidgetBuilderService
@description
Extends PermutationBuilderService for use with widgets.
Drives the business logic of the widget builder UI flow.

NOTE: injection returns an instance, not the constructor (see end of file).
###

angular.module 'app.widget'
.factory 'WidgetBuilderService', (
  PermutationBuilderService
  widgetFactory
  widgetStore
  $q
) ->
  class WidgetBuilderService extends PermutationBuilderService

    ###
    @name initialize
    @description
    Defines permutable attributes for widgets.
    ###

    initialize: ->
      super

      @permutable_attributes =
        name: []
        description: []

    ###
    @name buildPermutations
    @description
    Here, we extend the base method to ensure that we only build valid widgets.
    ###

    buildPermutations: ->
      super

      @permutations = _.filter @permutations, widgetFactory.validate

    ###
    @name createPermutations
    @description
    Specifies how a permutation gets persisted.

    @returns {Promise} - Fulfilled with the newly created permutations.
    ###

    createPermutations: ->
      # Grab the permutations before we empty them
      permutations = @permutations

      # Implement your AJAX call here
      widgetStore.addWidgets permutations

      # Reset our service
      @initialize()

      return $q.when permutations

  return new WidgetBuilderService()

```

[widget-builder-controller.coffee](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/widget/controller/widget-builder-controller.coffee)

```coffeescript
###
@name WidgetBuilderController
@description
Our controller and its template become a very thin layer that
glue the pieces together.
###

angular.module 'app.widget'
.controller 'WidgetBuilderController', (
  $scope
  $state
  WidgetBuilderService
) ->
  $scope.WidgetBuilderService = WidgetBuilderService

  ###
  @name submit
  @description
  Calls on the service to create permutations, then redirects to home page.
  ###

  $scope.submit = ->
    WidgetBuilderService.createPermutations().then (widgets) ->
      console.log 'look at all these widgets we built!'
      console.table widgets

      $state.go 'home'

```

[widget-builder.jade](https://github.com/kvcrawford/ng-permutation-builder/blob/master/src/widget/widget-builder.jade)

{% codeblock lang:jade %}
{% raw %}
.container-fluid
  .row
    .col-md-12
      p.lead Let's build some widgets.

  kc-permutation-builder(
    service = "WidgetBuilderService"
  )
    .row
      .col-lg-6.col-md-8.col-sm-6
          kc-permutable-input(
            name = "name"
            type = "text"
            required
          ) Name

          kc-permutable-input(
            name = "description"
            type = "text"
          ) Description

      .col-lg-4.col-md-4.col-sm-6
        .panel.panel-default
          .panel-heading
            h4.panel-title {{WidgetBuilderService.permutations.length}} Widgets Built

          .panel-body
            kc-permutable-attribute(
              name = "name"
            ) Name

            kc-permutable-attribute(
              name = "description"
            ) Description

          .panel-footer
            .btn-group
              button.btn.btn-primary(
                type = "button"
                ng-click = "submit()"
                ng-disabled = "WidgetBuilderService.permutations.length === 0"
              ) Submit

              button.btn.btn-default(
                type = "button"
                ng-click = "WidgetBuilderService.initialize()"
                ng-disabled = "WidgetBuilderService.permutations.length === 0"
              ) Reset

{% endraw %}
{% endcodeblock %}

## Going Forward
Now, it's trivial to implement a permutation builder for *Gadgets*. Or, say we wanted to support permutations of images? The surface area for changes needed is minimal: we just need a new `kcPermutableImage` directive, and the rest would work pretty much as-is.

Neat, huh?
