# Vue 2 &rightarrow; Vue 3 Migration Note

Here are major challenges I faced during migration from Vue 2 to Vue 3 of Buefy.

## Component alone cannot mount itself

A component instance no longer has the `$mount` method in Vue 3.
To mount a component, an app instance is necessary.

In Buefy, mounting a new component was utilized to create a component instance which is programmatically mounted; e.g., `modal.open`, `toast.open`.
One app instance can mount only one root component, so another app instance has to be created with the [`createApp`](https://v3.vuejs.org/api/global-api.html#createapp) API to mount a new component.
To programmatically mount a component, I changed the code to create a brand-new app instance as a mounting point of a new component.
The major drawback of this workaround is that a brand-new app **inherits no plugins**, thus **even components of Buefy itself cannot be used in a programmatically mounted component**.
As far as I investigated, there is no API to duplicate an app instance.
This will be especially problematic when a user wants to programmatically open a `Modal` with a custom component.
Components that a custom component depends on have to be explicitly specified in the `components` field.
See [#994da8e04b32fc2d2f75a1724903d64fdf206498](https://github.com/kikuomax/buefy/commit/994da8e04b32fc2d2f75a1724903d64fdf206498), and [#d68e2b42741b973d82df19f845ef95d628fc5447](https://github.com/kikuomax/buefy/commit/d68e2b42741b973d82df19f845ef95d628fc5447).

## No meaning of newing a component

Newing a component does not make sense on Vue 3.

Newing a component was utilized to create a default `Table` column renderer.
I made a new function `mockTableColumn` that creates an object that mocks the behavior of a `Table` column renderer.
See [#8f9d19255ca1e5c4eab9c2b6342864061c0d8dde](https://github.com/kikuomax/buefy/commit/8f9d19255ca1e5c4eab9c2b6342864061c0d8dde), and [#d5114f04fb2ec0f0db5b37aa942fb344e1293b6a](https://github.com/kikuomax/buefy/commit/d5114f04fb2ec0f0db5b37aa942fb344e1293b6a).

## Component cannot be obtained from a slot

As far as I investigated, no component instance can be obtained from a slot on Vue 3.

This was problematic for `Table` as it tried to list `TableColumns` specified in the default slot.
As a workaround, `TableColumns` registers itself to the parent `Table` when it is created, and removes it from the `Table` when it is unmounted.
See [#59b2dc901bfa87e84d0b5175780f0d5c228c714d](https://github.com/kikuomax/buefy/commit/59b2dc901bfa87e84d0b5175780f0d5c228c714d)

`ProviderParentMixin`, and `InejctedChildMixin` that are used by `Steps`, `Carousel`, etc. also faced this issue.
When a sorted flag is set, `ProviderParentMixin` indexes `InjectedChildMixin`s according to the index in the default slot.
I could not figure out how to enumerate component instances in the default slot.
So I introduced a new prop `order` to `InjectedChildMixin`, that is used to sort items.
See [#429b3fd6c205b2e8541aae0becd32e2b9a272857](https://github.com/kikuomax/buefy/commit/429b3fd6c205b2e8541aae0becd32e2b9a272857).

## `v-model` binding changed

Default `v-model` prop and event are changed,
- prop: `value` &rightarrow; `modelValue`
- event: `input` &rightarrow; `update:modelValue` 

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/v-model.html).

## No `$destroy`

There is no equivalent of [`$destroy`](https://vuejs.org/v2/api/#vm-destroy) in Vue 3.

This is problematic for Buefy, because a programmatically mounted component has to destroy itself.
I solved this problem by moving the responsibility for destruction from a component to an app instance created when the component is programmatically mounted (see the section [Component cannot be newed](#component-cannot-be-newed)).

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/introduction.html#removed-apis).

## Boolean attribute is not removed if `false`

On Vue 2, setting a boolean attribute to `false` is equivalent to removing it.
On Vue 3, setting a boolean attribute to `false` does not remove it, and `null` or `undefined` has to be specified to do so.

This caused some components look disabled.
I introduced computed values that become `true` or `undefined`, and associated them to boolean attributes instaed.

## Global `h` function

On Vue 3, a `render` function no longer takes a "create element" function, aka `h`, but the `h` function is [defined globally](https://v3.vuejs.org/api/global-api.html#h).
The second argument `props` of the global `h` API has been simplified; e.g., `staticClass` is merged to `class`, `on` is removed.

See also,
- [on{EventName} binding](#oneventname-binding)
- [Vue's migration guide](https://v3.vuejs.org/guide/migration/render-function-api.html).

## on{EventName} binding

### No `$listeners`

On Vue 3, there is no `$listeners` but it has been a part of `$attrs`.
A listener attribute name is "on" + event name.

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/listeners-removed.html).

### `h` function argument

On Vue 3, the second argument (`props`) of the `h` function handles a field whose name is "on" followed by an event name as event listener for the event.
For example, `onClick` listens for `click` event.

This causes conflict in props of the following components,
- `Dialog`
- `Modal`

They intended to take `onConfirm`, and `onCancel` props as callback functions independent from event listeners.
But they are interpreted as event listeners on Vue 3.
As a workaround, the callback functions are renamed to `confirmCallback`, and `cancelCallback` respectively.
See [#1a6784c3b5d1faa9f41d0d3cffe53604b15bda92](https://github.com/kikuomax/buefy/commit/1a6784c3b5d1faa9f41d0d3cffe53604b15bda92).

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/render-function-api.html#_2-x-syntax-3).

## Emitted events

On Vue 3, events emitted from a component has to be listed in the `emits` field.
Failing to list events may cause accidental binding of event listeners when `v-bind="$attrs"` is used.

For example, `Taginput` emits a `typing` event, and its child `Autocomplete` also emits a `typing` event.
If the `typing` event is not listed in the `emits` field of `Taginput`, a listener for `typing` events from `Taginput` will be called twice when a single `typing` event is emitted, one from `Taginput`, and the other from `Autocomplete`.
See [#447d2b3a6e6259c67b6949625271eea6ccb9ac98](https://github.com/kikuomax/buefy/commit/447d2b3a6e6259c67b6949625271eea6ccb9ac98).

## No directives

On Vue 3, the global `h` API no longer takes `directives`, use the global `withDirectives` API instead.
Vue 3 exports deicated objects representing built-in directives,

For example, `v-show` &rightarrow; `vShow`.

```js
import { vShow } from 'vue'
```

See [#9ca33f44ebd180832aa98ec2245ebfb805f1fbd2](https://github.com/kikuomax/buefy/commit/9ca33f44ebd180832aa98ec2245ebfb805f1fbd2), and [#6baf1b31c94a7d31491fdd97802bcfc11632d4ff](https://github.com/kikuomax/buefy/commit/6baf1b31c94a7d31491fdd97802bcfc11632d4ff).

See also [Vue's API reference](https://v3.vuejs.org/api/global-api.html#withdirectives).

## Directive hooks are renamed

Directives defined in Buefy had to be updated.
- `trapFocus` ([#74256c985c72113834b88a070a7f7d3212518020](https://github.com/kikuomax/buefy/commit/74256c985c72113834b88a070a7f7d3212518020))
- `clickOutside` ([#9ca33f44ebd180832aa98ec2245ebfb805f1fbd2](https://github.com/kikuomax/buefy/commit/9ca33f44ebd180832aa98ec2245ebfb805f1fbd2))

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/custom-directives.html).

## Functional component must be a function

On Vue 3, a single file component cannot be `functional`.

I had to rewrite `MenuList` as a function.
See [#612a0d24d8a361d95baa331918dc92b106dc707d](https://github.com/kikuomax/buefy/commit/612a0d24d8a361d95baa331918dc92b106dc707d).

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/functional-components.html#overview).

## No automatic component resolution

The `h` global API does not resolve a component name.
The [`resolveComponent`](https://v3.vuejs.org/api/global-api.html#resolvecomponent) global API has to be called.

Vue 3 exports dedicated objects for built-in components, `Transition` for example.

```js
import { Transition } from 'vue'
```

## Transition class names changed

As some transition class names have been changed in Vue 3, I had to update [src/scss/utils/_animations.scss](./src/scss/utils/_animations.scss).
See [#38393d9c5e28e036d137614c4e3caf731eaa1a22](https://github.com/kikuomax/buefy/commit/38393d9c5e28e036d137614c4e3caf731eaa1a22).

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/transition.html#_3-x-update).

## No `$children`

Vue 3 no longer provies `$children` of a component.

`Menu` depended on `$children` to list menu items, so I had to introduced a register/unregister mechanism to `Menu`.
See [#612a0d24d8a361d95baa331918dc92b106dc707d](https://github.com/kikuomax/buefy/commit/612a0d24d8a361d95baa331918dc92b106dc707d).

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/children.html#overview).

## No `_uid`

The internal field `_uid` of a component has been removed (moved?) in Vue 3.

`TabbedChildMixin` used to initialize its `value` prop with `this._uid`, and this no longer works (there is another reason that no `this` during prop initialization).
A Vue client code should not depend on an internal field anyway, so I introduced a new function `makeUniqueId` defined in a new file [`src/utils/make-unique-id.js`](./src/utils/make-unique-id.js), that returns a random number.
It depends on [`window.crypto.subtle`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/subtle).
See [#4bb8b78699911ba2b466468e4029ec0ec7f2dd55](https://github.com/kikuomax/buefy/commit/4bb8b78699911ba2b466468e4029ec0ec7f2dd55).

## v-for &times; ref does not list components

The special behavior of a `v-for` &times; `ref="X"` combination, where `refs.X` becomes an array of components `ref`ed by "X", is no longer supported on Vue 3.

As a workaround, I replaced "X" so that every component in a loop has a unique ref ID.
See [#f96494a54f21da16e1698d51e0a39c203b87fb89](https://github.com/kikuomax/buefy/commit/f96494a54f21da16e1698d51e0a39c203b87fb89).

## `inheritAttrs=false` affects `class` and `style`

On Vue 3, `class`, and `style` are members of `$attrs`, so setting `inheritAttrs=false` does not apply `class`, and `style` to the root element.

I had to manually copy `class`, and `style` to the root element.
See [#c39b77f708d360eed1f802440aae3f9d2cc79e81](https://github.com/kikuomax/buefy/commit/c39b77f708d360eed1f802440aae3f9d2cc79e81)

See also [Vue's guide](https://v3.vuejs.org/guide/component-attrs.html#disabling-attribute-inheritance).

## `$el` is `null` during beforeMount

See [#65cd34ce9a7e24e6ceadfae4241d1f965bf5ce3f](https://github.com/kikuomax/buefy/commit/65cd34ce9a7e24e6ceadfae4241d1f965bf5ce3f).

## Head scratchers

The following crashes made me scratch my head,
- Component in `<option>`'s `value`.
  On Vue 3, binding the `value` attribute of an `<option>` element to a component instance crashes.
  This affected `TableMobileSort` used in `Table` because it listed `<option>` elements whose `value`s are bound to individual component instances.
  As a workaround, `value` is bound to an index of a component instead.
  See [#59b2dc901bfa87e84d0b5175780f0d5c228c714d](https://github.com/kikuomax/buefy/commit/59b2dc901bfa87e84d0b5175780f0d5c228c714d).
- Execution timing of `$nestTick` changed?
  I had to replace a `$nextTick` call with a `setTimeout` call to wait for other events being processed.
  See [#82d9eeb96415e979efe888e427bcda469f861b19](https://github.com/kikuomax/buefy/commit/82d9eeb96415e979efe888e427bcda469f861b19).
- Accessing a slot that is never mounted. See [#b026667567fda6e4b67b71f25c2fffb56793cdfc](https://github.com/kikuomax/buefy/commit/b026667567fda6e4b67b71f25c2fffb56793cdfc).
- Maximum recursion errors. There were several reasons, and workarounds.
    - `v-if`, `v-else`, and `v-slot` in a single template. See [#d7e97bac2f55725961e72c660635b37fbfbd6bc2](https://github.com/kikuomax/buefy/commit/d7e97bac2f55725961e72c660635b37fbfbd6bc2).
    - Immediate call of watch methods? See [#5ada14b8eb63d1f5ce46b6e91d8671cbaa5ba04d](https://github.com/kikuomax/buefy/commit/5ada14b8eb63d1f5ce46b6e91d8671cbaa5ba04d).
    - Strange infinite updates. See [#124852a47cf9e3689c7706789c58f210e0caff4b](https://github.com/kikuomax/buefy/commit/124852a47cf9e3689c7706789c58f210e0caff4b).

## `$scopedSlots` is included in `$slots`

I had to replace many occurrences of `$scopedSlots`, but replacement was straightforward.

See also [Vue's migration guide](https://v3.vuejs.org/guide/migration/slots-unification.html#overview).

## beforeDestory is renamed to beforeUnmount

Updates were straightforward.

See [Vue's migration guide](https://v3.vuejs.org/guide/migration/introduction.html#other-minor-changes).