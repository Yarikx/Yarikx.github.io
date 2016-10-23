---
layout: post
title:  "Reductor - Redux for Android. Part 2: Composing reducers"
date:   2016-10-23
categories: Android Redux Reductor Redux.js
permalink: /Reductor-composition/
comments: true
image: http://www.tibco.com/blog/wp-content/uploads/2015/05/lego.jpg
description: Step by step tutorial how to use Reductor - Redux for Android
keywords: "Android, Java, Redux, Reductor, immutable, state"
---

{% include reductor-summary.md %}

# Recap

In [Part 1](/Reductor-introduction/) we covered what is [Reductor](https://github.com/Yarikx/reductor) and how to 
use it to model state transitions as pure functions.
We also used a feature of Reductor called "AutoReducer" to keep reducers type-safe and clean.

We ended up with reducer that looks like this:

{% gist Yarikx/27cb8eea028cc6a5bafc10635098ba9e %}

Today we are going to use our approach to storing multiple datasets in Reductor `Store`.

# Problem

In the previous article, we developed our Android app using Redux pattern. 
For our application, the instance of `Store` was a _Single source of truth_. 
That `Store` was holding and managing a single piece of data as a state -- `List<TodoItem>`.

But we live in real world where requirements change every day.
And as experienced android developers, we want to be able to add features to our application easily.

So what if we need to add one feature to our TODO application: allow the user to filter todo items by status.

As filter is just one of three options, we can model it as storing one of defined enum values:

{% gist Yarikx/122afa0a682d5a98c349e19264cd0b9a %}

But how can we store current filter? 
`Store` class is designed to store only one state object.

## Naïve approach

The simplest way that might come to mind is to create separate `Store` for `TodoFilter`.
To do that we would need to create one more `Reducer`. 
We will call it `FilterReducer`. 
Using [@AutoReducer](/Reductor-introduction/#reducing-boilerplate) it should be pretty straightforward.

{% gist Yarikx/80ec8ceac844b6aef87450853bcf392b %}

Then we create corresponding `Store`:

{% gist Yarikx/b46d99554044aebcfe78ba4cb6fdb9a8 %}

As it might look OK-ish initially, this idea is not as good as it seems.
By having two different stores we've lost the important property of our application -- *Single source of truth*.
It means that each time we want to dispatch the action, we need to know which store will handle it. 
That's not good.

Any other way? Yes!

# Composition

![Composition](http://www.tibco.com/blog/wp-content/uploads/2015/05/lego.jpg)

One of [3 principles of Redux](http://redux.js.org/docs/introduction/ThreePrinciples.html#single-source-of-truth) 
says that whole state should be represented as one tree with a single store.
That means in our case we need to do is to create one more class to hold both `List<TodoItem>` and `TodoFilter`.

{% gist Yarikx/ab1e02e8ca8b3e8fbf423e39339347a1 %}

Ok, nothing complex. One holder class to store both our sub-states.
The class is also immutable to prevent unexpected mutations (see [Redux principle 2](http://redux.js.org/docs/introduction/ThreePrinciples.html#state-is-read-only), [Part 0](/Reductor-prologue/)).

### Fixing reducer

But as we changed type our state from `List<TodoItem>` to `AppState`, reducer needs to be changed as well.
One can easily be created by merging `FilterReducer` and `TodoReducer` actions into one class.
Each action handler should also be updated to take and return `AppState`.

{% gist Yarikx/ead5b74fd9f475c9411a3a58f0618543 %}

That looks good. 
Now we can handle both actions related to `List<TodoItem>` and `TodoFilter`.
Even more, the structure with container class `AppState` allow us to add any arbitrary sub-states and action handlers for them.

However, this approach has some downsides:

* Particular sub-states need to be unpacked from `AppState` before processing, and then packed back to new `AppState`;
* When application will grow, reducer can become "God reducer" with all the actions we can perform on the whole state.

Let's see how we can sort it out.

# Refactoring

Our reducer now handles all the actions we have in our application.
But we still want logically separate actions that only relate to `List<TodoItem>` from actions that handle `TodoFilter`.
To do this we need to remind ourselves that `Reducer` is still a function with signature `(state, action) => state`.
And what we, as programmers are taught to do? -- Right! Compose functions.

If we will decompose our `AppStateReducer` to smaller functions to compute each sub-state if will look like this:

{% gist Yarikx/1e79f728629a500dbf3efd6af51f9d79 %}

This way we can manage how we can reduce each particular sub-state.
But hey, look closer at this function:

```java
List<TodoItem> items = reduceItems(state.todoItems, action);
```

Looks familiar, isn't it? Is it a... `Reducer`? Exactly!

Why not reuse reducer from the beginning of the article.
This way we can have our reducers small and concise.

{% gist Yarikx/a2dd9ea0077e78c4dbc90d35c0e7ec9b %}

That looks fancy. 

Our main reducer contains only the logic of dispatching actions to "sub-reducers" and combining the result.
That's the all it needs to know.

On the other hand, our "sub-reducers" only handle the logic related to particular "sub-state".
And they no nothing about the outer state.

That's what I call composition.

# Reducing boilerplate

What we have done is already good enough. 
But can we do it even better? Sure!

Take a look at our final `AppStateReducer` class.
Looks boring, doesn't it?

For the particular structure of state class, the structure of reducer will be pretty obvious.
But do you know any tool to help to write boilerplate code for us? -- Right! Annotation processor.

That's why [Reductor](https://github.com/Yarikx/reductor) 
has [@CombinedState](https://github.com/Yarikx/reductor#combine-reducers) annotation included.

To take advantage of annotation processor, we need to define our combined state as interface annotated with `@CombinedState`.
If our `AppState` will look like this,

{% gist Yarikx/c18556d8718742450f5fe22175116494 %}

reductor will generate the corresponding reducer:

{% gist Yarikx/863e26fee791c5d8eb116769fab52945 %}

The generated class will looks very similar to what we wrote by hand, except:

* Reducer includes `Builder` to make it more convenient to provide sub-reducers;
* Reducer does null-check on `state` to prevent null pointer exception (Why it matters I'll cover in one of the next articles);
* Before returning new state back, reducer do identity check for all the fields.
  And if all values are equals to original -- we can safely return current state.

Therefore instantiation of reducer will be straithforward:

{% gist Yarikx/c96d9a1daacdec24df337477802d8c46 %}

### Customizable **@CombinedState** class.

As you can see from the example, `AppState` is defined as an interface.
That means that Reductor will actually generate two things:

* Implementation of `AppState` (which will be called `AppStateImpl`);
* Implementation of `Reducer<AppState>` we saw before.

The implementation of our interface is just simple POJO, even without `toString()` and `equals()` defined.

{% gist Yarikx/50cc5069b8733dc62b378b9bcc014255 %}

But what if we need to make it `Serializable` or `Parcelable`?
What if we need to use this with Gson or just make `toString()` produce something human readable?
That's a big bunch of features to support.

However, there is already a project which specialises on generating value classes.
I'm talking about [AutoValue](https://github.com/google/auto/tree/master/value). 
If you not familiar with it, check out [Github page](https://github.com/google/auto/tree/master/value) and [documentation](https://github.com/google/auto/blob/master/value/userguide/index.md) for more information.

Since version `1.2` AutoValue supports extensions, so value classes can have suppot for 
[Parcelable](https://github.com/frankiesardo/auto-parcel), 
[Gson](https://github.com/rharter/auto-value-gson), 
[Moshi](https://github.com/rharter/auto-value-moshi) and much more.

And the good news is that Reductor support `AutoValue` classes as `@CombinedState` out of the box!
With AutoValue, our `AppState` will be defined as

{% gist Yarikx/ea06aa63868ccd6846c66b6c49e47dae %}

### Current **@CombinedState** support

At the moment ({{ page.date | date: "%b %-d, %Y" }}) Reductor support `@CombinedState` to be implemented either by

* Interface
* AutoValue class

Support for Kotlin data classes is planned.

# Conclusion

The idea of reducer as a pure function is powerful because of the ways it can be composed.
Keeping your reducers small and focused makes it's easy to manage and reason about them.

Taking this into account carrying all your application state tree in one object is just matter of combining smaller reducers.
And with help of Java annotation processor, we can make this even easier.
