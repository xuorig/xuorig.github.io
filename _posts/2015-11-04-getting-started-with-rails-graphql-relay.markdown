---
title:  "Getting started with Rails, Graphql and Relay (Part 1) "
date:   2015-11-11 11:00:00
description: How to get started with Facebook's Relay Framework using a Rails GraphQL API
---

For the past few weeks I've been playing with Facebook's [GraphQL][graphql] and [Relay][relay] for a personal project of mine. I thought I would give a quick overview of how to set up a Relay App using Rails as the API.

# What is [GraphQL][graphql] ?
Simply put, GraphQL is an application layer query language from Facebook. You can describe your API using a schema (a graph). All your clients can then query your data through the schema. GraphQL tries to solve one of the biggest problems we have with REST APIs these days: Changing data requirements in the clients.

Here's what a GraphQL Query would look like:
{% highlight javascript %}
{
  user(id: 1) {
    id,
    name,
    friends {
      name
    }
  }
}
{% endhighlight %}

The graphql response to this query would look like this:
{% highlight javascript %}
{
  "user" : {
    "id": 1,
    "name": "Marc-Andre Giroux",
    "friends": [
      {
        "name": "Joe Bro"
      },
      {
        "name": "Johny Yolo"
      }
    ]
  }
}
{% endhighlight %}

Now imagine we want to fetch the friends profile pictures, interests, and any other data. In we were using REST, we would either hit many other endpoints, or have to create a custom end point to fetch all that data at once.

With GraphQL, we only add what we want to the query and it will fetch it, just like this:

{% highlight javascript %}
{
  user(id: 1) {
    id,
    name,
    friends {
      name
      interests {
        name
      }
      profilePicture {
        url,
        width,
        height
      }
    }
  }
}
{% endhighlight %}

Check out the [Graphql Repo][graphqlrepo] for more information about GraphQL: How to query, mutate the schema and the type system.

# Introduction to relay

From Facebook's words, [Relay][relay] is a JavaScript framework for building data-driven [React][react] applications.

If you aren't familiar with React, take a look at [this][https://facebook.github.io/react/] before continuing.

### Containers

Relay introduces [RelayContainers][containers]. They compose your plain React components with the GraphQL query fragments which will fetch data that is specific to a component. It looks like this:

{% highlight javascript %}
class BlogPost extends React.Component {
  render() {
    var blogPost = this.props.blogPost;
    return (
      <View>
        <h1>{this.blogPost.title}</h1>
        <p>{this.blogPost.content}</p>
      </View>
    );
  }
}

// Export a Relay Container that wraps the original `BlogPost`, with the corresponding GraphQL fragment to describe the data requirements.
module.exports = Relay.createContainer(BlogPost, {
  fragments: {
    blogPost: () => Relay.QL`
      fragment on BlogPost {
        title,
        content
      }
    `,
  }
});
{% endhighlight %}

With this, each component now contains the data it needs with the UI and state.

### Routes

Routes can be a little confusing at first because of their name. They do not actually implement URL routing, instead they define an entry point into the Relay app. They define a GraphQL Root query that will be used to make the final query after putting together all the [fragments][frag] from the child components.


### Root Container

The [RootContainer][rootcontainer] basically puts the Routes and Containers together to construct the full GraphQL Query.

There's much more to it but it's time to start building something now that you know a bit more about GraphQL and Relay!

(For more detailed information about relay, please read the [docs][relaydocs]!)

This blog will be split in 2 parts. This part will be about building the GraphQL API and the second part about building our Relay app.

# Building the Rails GraphQL API

In this tutorial we're gonna build a simple Blogging app where you can fetch blogs and authors.

Lets start by building the API. Although it's possible to have both the relay app and the API live in a single Rails app, I chose to implement it as 2 different parts. Since the requests will be coming from a different origin than the API, we need to setup CORS. Lets use [rack-cors][rackcors] to do that.

First, add `rack-cors` to your `Gemfile`.
{% highlight ruby %}
  gem 'rack-cors', :require => 'rack/cors'
{% endhighlight %}

Add this code in `config/application`
{% highlight ruby %}
module MyBlogApp
  class Application < Rails::Application

    # ...

    config.middleware.insert_before 0, "Rack::Cors" do
      allow do
        origins '*'
        resource '*', :headers => :any, :methods => [:get, :post, :options]
      end
    end

  end
end
{% endhighlight %}

Obviously, don't actually use these settings in production, you will want to chose the allowed origins carefuly.

Next, we're gonna need some models for our Blog App. Lets create a Blog and an Author model

Run:
{% highlight ruby %}
rails generate model Blog title:string content:string author_id:integer
{% endhighlight %}

{% highlight ruby %}
rails generate model Author name:string
{% endhighlight %}

Our blog has one author so lets add that association to the model:
{% highlight ruby %}
class Blog < ActiveRecord::Base
  has_one :author
end
{% endhighlight %}


Alright it's time to bring GraphQL in. We're going to be using [rmosolgo's][rmo] [graphql-ruby][graphruby] which is an amazing gem to build our GraphQL Schema. Add the gem to your Gemfile.

{% highlight ruby %}
gem 'graphql'
{% endhighlight %}

Time to build our GraphQL Schema. Create a new folder in your app folder, `app/graph`. Everything GraphQL related will live there.

In that folder, create `app/graph/types`, where our custom graphql types will be created.

Make sure you add this line to your `application.rb` for rails to autoload the types:

{% highlight ruby %}
config.autoload_paths << Rails.root.join('app', 'graph', 'types')
{% endhighlight %}

As a first type, let's create the `Query` type. GraphQL always needs a query root for a schema. Create `query_type.rb` in `app/graph/types` and add the following code in it.

{% highlight ruby %}
# app/graph/types/query_type.rb
QueryType = GraphQL::ObjectType.define do
  name "Query"
  description "The query root for this schema"

  field :blog do
    type BlogType
    argument :id, !types.ID
    resolve -> (obj, args, ctx) {
      Blog.find(args[:id])
    }
  end
end
{% endhighlight %}

When declaring a type using the `graphql` gem, we have a few things to do. First we name the type (Query in that case). A small description is also needed for schema introspection. We declare fields on that type using the field method. Here we are using a block to define that field. We set the type for the `blog` field to BlogType and also give a resolve proc, which will be called when graphql wants to resolve that field. The proc takes 3 arguments, the obj (The query in that case), args which you will be able to pass in when you query, and context, a dict where you can pass any variables you might want to use on resolve time.

So, at the query root, we declare a field blog. which will return a blog by id. Let's add also the `BlogType` and the `AuthorType`

{% highlight ruby %}
# app/graph/types/blog_type.rb
BlogType = GraphQL::ObjectType.define do
  name "Blog"
  description "A Blog"
  field :title, types.String
  field :content, types.String
  field :author do
    type AuthorType
    resolve -> (obj, args, ctx) {
      obj.author
    }
  end
end
{% endhighlight %}


{% highlight ruby %}
# app/graph/types/author_type.rb
AuthorType = GraphQL::ObjectType.define do
  name "Author"
  description "Author of Blogs"
  field :name, types.String
end
{% endhighlight %}

Let's create the GraphQL Schema with our newly created types, at the root level of your graph folder, create this file:

{% highlight ruby %}
# app/graph/blog_schema.rb
BlogSchema = GraphQL::Schema.new(query: QueryType)
{% endhighlight %}

The only thing left now is exposing that Schema through an end point. The best way to do that is to handle POST requests with a graphql query as the data. Lets create `QueriesController` which will handle these POST requests.

{% highlight ruby %}
class QueriesController < ApplicationController
  def new
  end

  def create
    query_string = params[:query]
    query_variables = params[:variables] || {}
    result = BlogSchema.execute(query_string, variables: query_variables)
    render json: result
  end
end
{% endhighlight %}

And let's not forget to add a route to that controller:

{% highlight ruby %}
resources :queries, via: [:post, :options]
{% endhighlight %}


Time to test now! Add some mock data to your db and run queries using curl for now:

Query:
{% highlight bash %}
curl -XPOST -d 'query={ blog(id: 1) { title content }}' http://localhost:3000/queries
{% endhighlight %}

Response:
{% highlight bash %}
{"data":{"blog":{"title":"Intro to GraphQL","content":"Something something something. Blah blah blah. Etc etc etc."}}}
{% endhighlight %}


What if we wanted the author in the response ? Simply add it to the the query!

Query:
{% highlight bash %}
curl -XPOST -d 'query={ blog(id: 1) { title content author { name } }}' http://localhost:3000/queries
{% endhighlight %}

Response:
{% highlight bash %}
{"data":{"blog":{"title":"Intro to GraphQL","content":"Something something something. Blah blah blah. Etc etc etc.","author":{"name":"Marc-Andre Giroux"}}}}%
{% endhighlight %}

And that's it! We got a working GraphQL API using Ruby on Rails. __Stay tuned for the next part__ where we will build an app using React/Relay using the API we just built.

You can take a look at the [source here][src].

#### If you have more questions about GraphQL, come hang out in Slack! [https://graphql-slack.herokuapp.com/][slack]

#### Edit:
As Sergio del Rio pointed out in comments, you'll want to change `protect_from_forgery` to `null_session`
in your `ApplicationController` for your api to work correctly at first. Also make sure you name your files the same as your class name for Rails to load them correctly. `QueryType` -> `query_type.rb`

#### Edit2:
Instead of using curl to test your queries, you can use GraphiQL, an amazing tool to test and even autocomplete your GraphQL queries. [Here's how to install it in your Rails App.][install]

You can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[app]: https://github.com/xuorig/my-simple-blogging-app
[graphql]: http://facebook.github.io/graphql/
[relay]: https://github.com/facebook/relay
[graphruby]: https://github.com/rmosolgo/graphql-ruby
[rmo]: https://github.com/rmosolgo
[graphqlrepo]: https://github.com/facebook/graphql
[react]: https://facebook.github.io/react/
[containers]: https://facebook.github.io/relay/docs/guides-containers.html
[rootcontainer]: https://facebook.github.io/relay/docs/guides-root-container.html#content
[relaydocs]: https://github.com/facebook/relay
[install]: http://mgiroux.me/2016/setting-up-graphiql-with-rails/
[rackcors]: https://github.com/cyu/rack-cors
[slack]: https://graphql-slack.herokuapp.com/
[src]: https://github.com/xuorig/my-simple-blogging-app
[https://facebook.github.io/react/]: https://facebook.github.io/react/
