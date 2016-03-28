---
title:  "Implementing the Relay spec in a GraphQL server"
date:   2016-03-27 10:00:00
description: The difference between a vanila GraphQL server and a Relay compatible one, and how to implement the spec. 
---

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@__xuorig__">
<meta name="twitter:creator" content="@__xuorig__">
<meta name="twitter:title" content="Implementing the Relay spec in a GraphQL server">
<meta name="twitter:description" content="The difference between a vanila GraphQL server and a Relay compatible one, and how to implement the spec. 
">
<meta name="twitter:image" content="data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI2MDAiIGhlaWdodD0iNjAwIiB2aWV3Qm94PSIwIDAgNjAwIDYwMCI+PGcgZmlsbD0iI0YyNkIwMCI+PHBhdGggZD0iTTE0Mi41MzYgMTk4Ljg1OGMwIDI2LjM2LTIxLjM2OCA0Ny43Mi00Ny43MiA0Ny43Mi0yNi4zNiAwLTQ3LjcyMi0yMS4zNi00Ny43MjItNDcuNzJzMjEuMzYtNDcuNzIgNDcuNzItNDcuNzJjMjYuMzU1IDAgNDcuNzIyIDIxLjM2IDQ3LjcyMiA0Ny43MiIvPjxwYXRoIGQ9Ik01MDUuMTggNDE0LjIyNUgyMzguMTI0Yy0zNS4yNSAwLTYzLjkyNi0yOC42NzQtNjMuOTI2LTYzLjkyM3MyOC42NzgtNjMuOTI2IDYzLjkyNi02My45MjZoMTIwLjc4YzIwLjgxNiAwIDM3Ljc1My0xNi45MzggMzcuNzUzLTM3Ljc1NnMtMTYuOTM4LTM3Ljc1Ni0zNy43NTMtMzcuNzU2SDk0LjgxYy03LjIyNyAwLTEzLjA4Ni01Ljg2LTEzLjA4Ni0xMy4wODUgMC03LjIyNyA1Ljg2LTEzLjA4NiAxMy4wODUtMTMuMDg2aDI2NC4wOTNjMzUuMjUgMCA2My45MjMgMjguNjc4IDYzLjkyMyA2My45MjZzLTI4LjY3NCA2My45MjMtNjMuOTIzIDYzLjkyM2gtMTIwLjc4Yy0yMC44MiAwLTM3Ljc1NiAxNi45MzgtMzcuNzU2IDM3Ljc2IDAgMjAuODE2IDE2LjkzOCAzNy43NTMgMzcuNzU2IDM3Ljc1M0g1MDUuMThjNy4yMjcgMCAxMy4wODYgNS44NiAxMy4wODYgMTMuMDg1IDAgNy4yMjYtNS44NTggMTMuMDg1LTEzLjA4NSAxMy4wODV6Ii8+PHBhdGggZD0iTTQ1Ny40NjQgNDAxLjE0MmMwLTI2LjM2IDIxLjM2LTQ3LjcyIDQ3LjcyLTQ3LjcyczQ3LjcyIDIxLjM2IDQ3LjcyIDQ3LjcyLTIxLjM2IDQ3LjcyLTQ3LjcyIDQ3LjcyLTQ3LjcyLTIxLjM2LTQ3LjcyLTQ3LjcyIi8+PC9nPjwvc3ZnPg==">

I've seen some confusion on Relay and GraphQL lately. GraphQL is so often used with Relay that I think sometimes we forget what a GraphQL server is, and what Relay adds on top of it. The goal of the next 3 posts is to try to clearly see the line between the two, and how to implement the Relay part in an existing GraphQL Server.

### Relay

From Facebook's words Relay is simply a 

> Framework for building data-driven React applications.

It is a wrapper around Relay components, enabling them to live next to the GraphQL queries they need to fulfil their data requirements.
On the client side, Relay's strength is it's amazing client side cache. On the server side, Relay expects the GraphQL server to implement a certain spec for things to work perfectly.

### GraphQL & Relay

Let's take a look at what a Relay app needs from a GraphQL server that is different from usual. First, the _Global Object Identification_. Relay introduces the need for objects with unique identifiers, called "nodes". With that concept, Relay can refetch arbitrary objects from a GraphQL server using their GID. On the GraphQL server, each object must have an `id` field returning that `GID`, and the root query type must have a `node` field, used to query nodes globally.

Next, _Connections_. Connections are an Object that provide a standard way to slice and paginate results. Connections can provide `PageInfo`, a way of telling the client if there are more results to fetch still. The Connection object provides a cursor for every item in it's result set.

The last thing that Relay adds to a GraphQL server is the _Input Object Mutation_. Compared to regular GraphQL mutations, Relay mutations are similar, but try to standardize the way they are exposed and called. Every mutation takes 1 argument, an Input Object. This input object must always contain a `clientMutationId`, and the GraphQL server must always return it in the response.

So as you see, 3 things are added on top of a vanila GraphQL server:
  - Global Object Identification
  - Connections
  - Input Object Mutations

### Implementing the spec 

Each of these require quite a bit of work to implement in a GraphQL server. I've done the exercise myself, and will share what I've done to  implement the Relay spec in a Ruby on Rails GraphQL server, using the `graphql` gem as a base. I will divide the results in 3 blog posts, starting with this one. Let's start by the `Global Object Indentification`.

### Global Object Indentification

Let's start by making our current Object Types compatible with Relay. If you remember from above, each object had to have an `id`field. GraphQL has support for interface objects, this is a great use case for what we want to do. When using interfaces in the `graphql` gem, the object implementing it "inherits" from it's field. 

So the first step is to create that interface type, let's call it `Node`.

{% highlight ruby %}
# lib/graph/relay/node.rb
module Graph
  module Relay
    Node = GraphQL::InterfaceType.define do
      name "Node"
      field :id, !types.ID, property: :to_global_id

      resolve_type -> (object) do
        Graph::Relay::Node.possible_types.detect do |type|
          object.is_a?(type.model)
        end
      end
    end
  end
end
{% endhighlight %}

This should not look too unfamiliar to you. We're simply defining an Interface type using the `graphql` gem. An interface must be able to resolve the type of an object. The code is pretty straight forward, we look through all of the `possible_types` and find one that matches our object using `type.model`.

Our field here, will simply resolve by calling `object.to_global_id`. We need Global Identification, so I used something that Rails has by default: [GlobalID][globalid]. Calling `object.to_global_id` transforms your ActiveRecord into a `GlobalID` oject, a unique identifier for that particular record.

{% highlight ruby %}
=> User.find(1).to_global_id
=> #<GlobalID:0x007f9abca6ecb8 @uri=#<URI::GID gid://app-name/User/1>>
{% endhighlight %}

You might've noticed that `type.model` is not an attribute that types have using the `graphql` gem. Since we're using rails, I've augmented the normal GraphQL object type with my own attributes, like this:

{% highlight ruby %}
# lib/graph/active_record_object_type.rb
module Graph
  class ActiveRecordObjectType < GraphQL::ObjectType 
    attr_accessor :model
    accepts_definitions(:model)
  end
end
{% endhighlight %}

`accepts_definitions` is a new addition to the `graphql` gem. It basically lets you add custom fields to an ObjectType, while still being able to use the DSL `graphql-ruby` provides.

With that being done, we can simply implement this interface in our existing fields, just like this: 

{% highlight ruby %}
# app/graph/user_type.rb
UserType = Graph::ActiveRecordObjectType.define do
  name "User"
  description "A User"
  interfaces [Graph::Relay::Node]
  model User

  field :email, !types.String, property: :email
end
{% endhighlight %}

The two lines that interest us here are `model User` which will indicate which ActiveRecord model our type is refering to, and `interfaces [Graph::Relay::Node]`, which does exactly what we wanted to do!

There's now only one thing left to do. Exposing a `node` field on the query root. We can do this just like defining any other field:

{% highlight ruby %}
  field :node do
    type Graph::Relay::Node
    argument :id, !types.ID

    resolve -> (object, args, _) do
      gid = GlobalID.parse(args["id"])
      GlobalID::Locator.locate(gid)
    end
  end
{% endhighlight %}

Our field accepts one argument, `id`. And uses `GlobalID::Locator` to get the corresponding object, pretty simple!

Lets test this!


First, let's see if the object we query have the `id` field available:

`query getObjectId { viewer { currentUser { email } } }`

<img width="637" alt="screenshot 2016-03-27 19 07 34" src="https://cloud.githubusercontent.com/assets/1919498/14068402/54fe279a-f450-11e5-8726-7520882996a6.png">

Then, we should be able to query any object using the `node` field:

`query getObjectById { node(id: "gid://review-buddy/User/1") { ... on User { email } } }`

<img width="602" alt="screenshot 2016-03-27 19 12 10" src="https://cloud.githubusercontent.com/assets/1919498/14068401/53951aee-f450-11e5-8c00-e9b0c531ec4a.png">



Next week I'll have a post up about implementing the second thing Relay needs from a GraphQL server: _Connections_
 
As always you can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[globalid]: https://github.com/rails/globalid
[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig


