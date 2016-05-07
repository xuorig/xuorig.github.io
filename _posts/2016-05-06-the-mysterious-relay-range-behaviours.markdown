---
title:  "RANGE_ADD mutations and the mysterious Relay Range Behaviors"
date:   2016-05-06 10:00:00
description: Demystifying Relay Range Behaviors when using RANGE_ADD mutations
---

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@__xuorig__">
<meta name="twitter:creator" content="@__xuorig__">
<meta name="twitter:title" content="The mysterious Relay Range Behaviors">
<meta name="twitter:description" content="Demystifying Relay Range Behaviors when using RANGE_ADD mutations">
<meta name="twitter:image" content="https://cloud.githubusercontent.com/assets/2231765/9094460/cb43861e-3b66-11e5-9fbf-71066ff3ab13.png">

One of the coolest features of Relay is it's amazing client side cache. When we mutate our data using Mutations, Relay needs to know how to update it's client cache. The way the Relay team chose to manage that is by having `Mutator Configurations`. The `Mutator Configuration` is a way to instruct Relay what to do with the Mutation payload, AKA how to to update it's client side cache with the serve response. There is currently 5 ways to use this `Mutator Configuration`:

  - `REQUIRED_CHILDREN`
  - `FIELDS_CHANGE`
  - `NODE_DELETE`
  - `RANGE_DELETE`
  - `RANGE_ADD`

### RANGE_ADD Mutations

Most apps will eventually want to add some item to some collections, and you'll usually end up using the `RANGE_ADD` configuration, this post will be strictly about `RANGE_ADD` and some of it's gotchas. I have contributed a bit on mutations, especially `RANGE_ADD` so I thought it would be a good idea to share some new things about this!

First let's take a look at what it looks like! Let's take the famous TODO app example. We want to create a Mutation that will add a new Todo item to our list:

{% highlight javascript %}
export default class AddTodoMutation extends Relay.Mutation {
  getMutation() {
    return Relay.QL`mutation{addTodo}`;
  }

  getFatQuery() {
    return Relay.QL`
      fragment on AddTodoPayload {
        todoEdge,
        viewer {
          todos,
          totalCount,
        },
      }
    `;
  }

  getConfigs() {
    return [{
      type: 'RANGE_ADD',
      parentName: 'viewer',
      parentID: this.props.viewer.id,
      connectionName: 'todos',
      edgeName: 'todoEdge',
      rangeBehaviors: {
        '': 'append',
        'status(any)': 'append',
        'status(active)': 'append',
        'status(completed)': 'ignore',
      },
    }];
  }

  getVariables() {
    return {
      text: this.props.text,
    };
  }
}
{% endhighlight %}

Take a look at the `getConfigs` function above, this is the `Mutator Configuration` for that Mutation. We specify the type (RANGE_ADD), the parent, connection, the new edgeName that we expect to receive from the server and the `rangeBehaviors`. What is are rangeBehaviors? It's a way of telling Relay what you want to do with that new edge.

#### Range Behavior values

Previously the only two choices we had were either `append` or `prepend`. As you might've guessed, these were telling Relay to either `append` the new edge to the connection in the cache, or to prepend it. 

What if you didn't want to do anything with the new edge ? You could pass `null` instead, but it was not the cleanest way to do it. 

Wanted to actually refetch the whole connection and trust the server instead of updating the client cache with a single new edge ? No real way of doing it, except simply not adding an entry in the `rangeBehaviors` object. Unfortunatly that would result in a warning, telling you that Relay refetched the whole connection, even though maybe that was the desired behaviour.

Since then, two new range behaviors were introduced, `ignore` and `refetch`. `ignore` is  meant to replace the `null` to instruct Relay not to do anything with the new edge and `refetch` is meant to squelch the warning Relay would throw when not including a range in the `rangeBehaviors` object.

Alright, we know which values to use in our `rangeBehaviors`, but what should I put in the key ? The keys allow you to define different behaviors for different connections. Take a look at the `rangeBehaviors` from the previous example:

{% highlight javascript %}
rangeBehaviors: {
  '': 'append',
  'status(any)': 'append',
  'status(active)': 'append',
  'status(completed)': 'ignore',
}
{% endhighlight %}

We have an empty string `''`, `status(any)`, `status(active)`, and `status(completed)`. To understand why we have to format our `rangeBehaviors` this way, we have to understand how Relay matches a connection to a `rangeBehavior`!

When trying to get a `rangeBehavior` for a particular connection, Relay first gets the `rangeFilterCalls`. `rangeFilterCalls` are simply an Array containing what the connection was called with. Keeping the same example, if we queried the todos by `status: 'any'` we would have these `rangeFilterCalls`:

{% highlight javascript %}
[{name: 'status', value: 'any'}]
{% endhighlight %}

Now, to match a key with these calls, Relay transforms these calls by serializing them into a nice string such as `status(any)`, making it way easier to use it as a key.

It `maps` over each call and serializes the call this way: `.name(value)`. If we were to have these calls:
 
{% highlight javascript %}
[{
  name: 'status',
  value: 'any'
}, {
  name: 'someArg',
  value: 'someValue'
}]
{% endhighlight %}

We would end up with this key: `someArg(someValue).status(any)` (Notice that rangeFilterCalls are sorted and that the first `.` is removed!)

For this reason, you have to make sure to include all calls when defining your `rangeBehaviors` oobject. Using the same calls, a range behavior of `someArg(someValue): 'append'` would trigger a warning since we forgot to also include the `.status(any)`.

Now, you'll notice it's not always possible to choose the best behavior given only the serialized key. Sometimes we want to have `append` when `someArg` has `someValue`, but only when `status` is `complete`. We had no way of doing this before recently.

#### Defining rangeBehaviors as a function

I recently submitted a PR to have another way to define `rangeBehaviors`. As I'm writting this, this feature has been merged on `master` but is not released yet (https://github.com/facebook/relay/pull/1054)

Here's how our Todo example looks with a `rangeBehaviors` function instead of the plain object.

{% highlight javascript %}
rangeBehaviors: (calls) => {
  if (calls.status === 'completed') {
    return 'ignore';
  } else {
    return 'append';
  }
}
{% endhighlight %}

When we define `rangeBehaviors` as a function, Relay won't serialize the calls but simply pass them to the function. We can then implement any logic needed to decide on the proper behavior for this connection/calls. Much better right ?

Time to try it yourself! Let me know how it goes :)

As always, you can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig

