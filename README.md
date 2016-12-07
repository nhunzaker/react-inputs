# React Inputs

Hey everyone! Tonight I'm going to talk a little bit about inputs in
React. Specifically controlled inputs.

React is a Virtual DOM library, which means that it builds up an ideal
idea of what the user interface should look like, then it reconciles
differences to translate the current state into that ideal state.

Of course, this is kind of troublesome for form data. Form data is
inherently stateful, the DOM keeps track of a user's interactions with
a form through properties on HTML elements.

This is problematic for React's idea of keeping state contained in
React. So it takes a lot of steps to ensure that it controls form
state as much as it can to avoid edge cases.

## What we'll talk about

I only have 15 or 20 minutes today, so I won't be able to go into
every single detail of how React manages controlled inputs, but I did
want to cover a few items:

1. Mount
2. Post mount
3. defaultValue vs value
4. value "detachment"
5. Wrap up!

## Mount

I'm going to start at the mounting process. The process in which a
virtual DOM node gets converted into something real.

### mountWrapper

```
react/src/renderers/dom/shared/ReactDOMComponent.js
512: mountComponent()
```

After assigning some member properties, mountComponent switches over
the element's tag name. Every input is "wrapped" using
ReactDOMInput. This bootstraps the input and conducts some
validations. Let's step into that.

```
react/src/renderers/dom/client/wrappers/ReactDOMInput.js
512: mountWrapper
```

So before anything, in development mode, React goes through quite a
few validations to confirm proper usage of inputs. It warns about
deprecations, like the LinkedState behavior. Then it makes sure that
the controlled input's defaultValue and value props are configured
correctly.

### defaultValue and value

Let's break out from code review for a minute to talk about
`defaultValue` and `value`.

```
examples/default-value.html
```

So I think we're all a familiar with this code. We use `value` when
working with a controlled input, `defaultValue` when working with an
uncontrolled input.

When I first came to React, I assumed that `defaultValue` was a
React thing. This isn't true!

`defaultValue` and `value` are both input properties. The difference
is that `defaultValue` aligns with the `value` attribute. This is the
value you set in normal HTML. The initial value for the input

`value` represents the active value. The value managed by the
user. Assigning value changes what you see on the screen (side from a
few cases that we'll talk about in a minute).


### Back to ReactDOMInput

```
react/src/renderers/dom/client/wrappers/ReactDOMInput.js
146: mountWrapper
```

Okay! So moving along. The last thing in this function that I want to
highlight is this `wrapperState` business. It stores some information
about the input that is important for controlled values. It assigns an
initial value, so if value is ever null, it knows what to fall back
to, and it sets up a custom change event.

I'll get to the custom change event later. For now, just remember that
it's a special change event for controlled inputs that helps keep an
input in sync with React.

```
react/src/renderers/dom/shared/ReactDOMComponent.js
541: mountComponent()
```

### Getting Props

Okay. We're done here. Now let's go back to ReactDOMComponent. The
next thing that happens is that we get the initial properties for the
component. Inputs go through a special filter: getHostProps.

```
react/src/renderers/dom/client/wrappers/ReactDOMInput.js
58: getHostProps
```

Okay. There's some special stuff going on here.

First, we need to always make sure that certain attributes are
assigned first. `type`, `step`, `min`, and `max` get assigned first,
because they can influence the value property. If they don't get set
first, value can change in unexpected ways in IE11.

Second, React technically still supports the `LinkedValue` plugin. To
accommodate it, React overrides the value prop with the result of
special logic inside of `LinkedValueUtils`. For most purposes, it just
returns props.value.

Third, remember that special onChange we looked at earlier? Here's
where it gets used. All inputs get the same onChange handler that is
internal to React. Later, we'll see when the `onChange` handler _you_
define gets called.

## Post Mount

Okay! We're moving along now! Jumping back to ReactDOMComponent. After
some DOM creation bits, where we generate markup and set properties,
we hit another tag name switch:

```
react/src/renderers/dom/shared/ReactDOMComponent.js
649: mountComponent()
```

First thing we see is this postMount behavior. This ties directly into
`ReactDOMInput.postMount`:


```
react/src/renderers/dom/client/wrappers/ReactDOMInput.js
224: postMountWrapper
```

Okay. Let's dive in. First we get the props for the current _element_
associated with the component instance. Then we get the actual DOM
node from a map of component instances to real world HTML elements.

There's an extra hangup here. Based on the type of input given, React
directly assigns `value` and `defaultValue`. This process is called
detachment. Let's dive into that now.

## Detachment

Detachment is sort of confusing, so I made a step by step guide for
how it works:

1. Before value is ever touched, assigning defaultValue influences the
   value property.
2. As soon as value is touched, either by JavaScript or a user,
   defaultValue stops influencing the display value.

Lets see what that looks like

```
http://localhost:4000/examples/detachment.html
```

So we can see that editing defaultValue up until the user interacts
with it, or the value attribute is assigned, sets the value. This is
really important for React. We don't want changes to the defaultValue
attribute to influence the display value.

Additionally, some input types, like date and color pickers, have
display issues if value isn't intentionally in a specific way. Hairy,
something React does for us. Otherwise there is a display glitch where
the value doesn't show up at all!

Finally, there's some silliness around checkboxes, but we won't get
into that.

## Change events

So now we've gotten that far. But the time this function finishes
executing, React will have put the input on to the page and attached
event listeners.

So what happens when things _do_ change? Recall that every
ReactDOMInput has a custom onChange handler. This gets called as the
result of some complicated logic that normalizes change events in
React. It manipulates the `onChange` event to trigger on every
possible update to the `value property`.

Let's quickly run through that now.

Whenever a change event fires, it runs through React's event plugin
hub:

```
react/src/renderers/shared/stack/event/EventPluginHub.js
225: ExtractEvents
```

Most events in React are delegated, unless browsers don't play
nice. Events bubble up to the top of the DOM tree and trigger a
generic event listener triggered by React.

When this happens, React uses a module called EventPluginHub to
process the event. It runs through a sequence of plugins that
normalize browser behaviors. This is the synthetic event system you
may have heard of.

If we dig into the ChangeEventPlugin, we can quickly see all of the
work React does to normalize change events for us:

```
react/src/renderers/dom/client/eventPlugins/ChangeEventPlugin.js
All of it
```

This plugin is a whole talk unto itself. Just know that React is doing
a lot of work to make this happen under the hood.

What matters to us is that this eventually calls that custom
`onChange` function we looked at earlier. If you dig into React
master, this has actually gone away, but this talk is current as of
15.4.1, and it's a necessary step along the way:


```
react/src/renderers/dom/client/wrappers/ReactDOMInput.js
275: ReactDOMInput
```

We're almost there. I promise. Most of this function actually relates
to controlling radio buttons. Don't worry about that. The important
thing is:

React calls LinkedValueUtils to invoke the onChange handler. If you
aren't using LinkedValueUtils, this essentially invokes onChange
directly. If we have time, I can dig into it specifically.

This is going to call onChange prop given to the input, which will
likely result in a state change. Since state changes, it'll cue up a
re-render of the UI with the new value.

```
react/src/renderers/dom/shared/DOMPropertyOpertions.js
133: setValueForProperty
```

The logic is a little hard to parse. This is the part that
matters. React sets the `value` _attribute_. This is important for
preventing the `form.reset()` from reverting an input to it's
defaultValue. This doesn't trigger a change event, so if the value
attribute isn't set, React can get out of sync with the DOM.

Of course, we also need to update the value _property_. To do that, we
call one last function: `updateWrapper`

## updateWrapper

```
react/src/renderers/dom/client/wrappers/ReactDOMInput.js
158: updateWrapper
```

This is how the value property actually gets updated by React. Let's
skim past the validations and focus on what really matters.

We know we have a controlled input if a value prop is present. For
these cases, assign the new value. However only mutate the value
property if it is different. If we blindly updated the value property
on every change, text selection jumps to the end of the input because
the browser gets confused.

As an asside, defaultValue, we go through a similar process.

## We made it

Okay. That's it! Just to recap:

1. An input mounts to the page, setting up initial props required for
   controlled inputs.
2. After it mounts to the page, it runs through some extra steps to
   detach the value property from the value attribute. This fixes
   some edge cases
3. When an input changes, it runs through a special event plugin for
   change events.
4. This eventually calls our change event, and then issues a command
   to sync up the value attribute and value property to avoid issues
   where controlled inputs might get out of sync with React.

## Wrapping up

Yay! Thank you for listening. This was a lot of code, and there are
many steps involved, can I answer any questions?
