---
title:  "My journey into GraphQL-Ruby's query execution"
date:   2016-02-14 11:00:00
description: Overview of how GraphQL-Ruby resolves GraphQL queries using it's Executor
---

`schema.execute()` is the public method we all use to execute graphql queries on our graphql-ruby powered Apis currently. But what actually happens when we call that method ? A cool thing `graphql-ruby` allows us to do is create our own query executors to use with the gem. I went on a journey into the default executor used by `graphql-ruby` and here are my finding:

### It all starts at `my_schema.execute()`

#### `GraphQL::Query`

All `schema.execute` does is create a `GraphQL::Query`, passing itself to the query and then calling `query.result`.

The `GraphQL::Query` object will validate any errors in the query right away, early returning any errors found.

In the case where there are no validation errors, the query object will use a `GraphQL::Query::Executor` the get the query results.

#### `GraphQL::Query::Executor`

The executor is in charge of executing an operation `query | mutation | subscription` on a graphql query. It also rescues any error that could happen during the execution.

It will first find the `selected_operation`. You can technically have multiple operations in a graphql_query, but we it only execute one.

With the `selected_operation` found, we can finally get the `RootType` for our operation. `graphql-ruby` calls `query.schema.public_send(operation_type)`. We'll also need to get a special executor for that `operation_type`, so we call the schema to get the execution strategy too. (This is where you could set up your own strategy on the schema).

Now that we have the `RootType` and the `execution_strategy` we need, let's get their result!

#### `execution_strategy.execute(operation, root_type, query)`

For now, the `execution_strategy` for all operation types is always `SerialExecution`

#### `GraphQL::Query::SerialExecution`
> aka GraphQL::Query::BaseExecution

SerialExecution is pretty much a wrapper for the three main classes we will need to resolve our query:

{% highlight javascript %}
  GraphQL::Query::BaseExecution::FieldResolution
  GraphQL::Query::BaseExecution::OperationResolution
  GraphQL::Query::BaseExecution::SelectionResolution
{% endhighlight %}


From the spec:

> When evaluating a grouped field set serially, the executor must consider each entry from the grouped field set in the order provided in the grouped field set. It must determine the corresponding entry in the result map for each item to completion before it continues on to the next item in the grouped field set:

We were still at the beggining of our execution. So obviously we will be going to start by resolving our operation, the top level of our query. We pass it the `ast_operation`, `root_type`, `query_obj`, and `self`.

#### `GraphQL::Query::BaseExecution::OperationResolution`

Operation resolution is pretty straight forward. We take all the selections on the query, we instanciate a `SelectionResolution` object, pass it all the selections and let it do the rest of the job!

#### `GraphQL::Query::BaseExecution::SelectionResolution`

Before we go deeper, let's mke sure we understand Operations, Selections and Fields. Take a look at this query where we fetch heroes of two starwars movies.
Our `operation_type` is `query` and our `operation_name` that will be the selected operation here would be `starwars`.

{% highlight javascript %}
query starwars {
  heroNewHope: hero(episode: NEWHOPE) {
    id
    name

  }
  heroJedi: hero(episode: JEDI) {
    id
    name
  }
}

{% endhighlight %}
The top level selections on this query would look something like this

{% highlight javascript %}
alias:heroNewHope, name=hero, selections=[id, name]
alias:heroJedi, name=hero, selections =[id, name]
{% endhighlight %}

Ok. This is where more things happen, in the `SelectionResolution`, we first flatten all the selection, and merge all fields to avoid resolving the same thing more than once, that means that if we have something like this:

{% highlight javascript %}
fragment DuplicateFields on Character {
  id
  name
  appearsIn
}

query starwars {
  heroNewHope: hero(episode: NEWHOPE) {
    id
    name
    ...DuplicateFields
  }
}
{% endhighlight %}

we'll first merge everything to this instead:

{% highlight javascript %}
query starwars {
  heroNewHope: hero(episode: NEWHOPE) {
    id
    name
    appearsIn
  }
}
{% endhighlight %}

This is also where we will apply directives like these:

{% highlight javascript %}
    ...DuplicateFields @include(if: true)
{% endhighlight %}

And `graphql-ruby` uses a `DirectiveChain` to do that.

`DirectiveChain` is a simple class taking a block that will resolve a fragment. Using the `directive.resolve` proc, it determines if the fragment should be skipped/included or not.

When we've flattened the selections, and passed them thorugh the directive chain, its time to resolve the fields.

#### `GraphQL::Query::BaseExecution::FieldResolution`

FieldResolution is where we finally call the `resolve` function of our Field.

A fun thing that `graphq-ruby` adds to the `GraphQL` spec, is being able to use middlewares. Middlewares are applied right before actually resolving, and can be used to do anything you want with a field before its being resolved, including halting the execution on that field.

##### `get_raw_value`
Calls every step of the middleware chain and finally calls the resolve proc using `field_definition.resolve(parent, args, context)`.

If the value we get is `GraphQL::Query::DEFAULT_RESOLVE`, a `public_send` with the field name is sent to the parent object.

##### `get_finished_value`
In `get_finished_value`, we take the value we got from `get_raw_value`, return if its `nil`, or an `ExecutionError`. If we actually have a value, we use `GraphQL::Query::BaseExecution::ValueResolution` to keep executing the query, on the value.

### Value Resolution

Value Resolution defines different classes to resolve every GraphQL type such as:

#### Object

In the case of an Object type, we know we'll have selections on it. So we get our `selection_resolution` and resolve these again.

#### List

When resolving a List type, we simply get the underlying type of the items, and resolve each of them individually.

#### Enum and Scalar

If we get a Scalar or an Enum, we know we're at a the end of a patg and that we can get a value. `coerce_result` is called on the field to get the value. `coerce_result` is simply a proc used to manipulate the final result value.

The proc could look like something like this for example:

{% highlight javascript %}
coerce_result -> (value) { value.to_f }
{% endhighlight %}

#### NonNull

For a `NonNull` type, kind of like with a `List` type, we get the underlying type and resolve it with the proper strategy.

When all selections on the top level operation are resolved, we finally return the Hash containing the entire query resolved.

